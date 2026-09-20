---
title: "How I Built a MikroTik Dual-Uplink PCC Load Balancing Network RADIUS server enabled"
date: 2026-09-20
draft: false
tags: ["mikrotik", "routeros", "pcc", "load-balancing", "hotspot", "networking", "failover", "wifi"]
description: "An in-depth, reusable guide for deploying a MikroTik RouterOS 7 dual-uplink PCC load-balancing network with recursive failover, HotSpot, NAT, policy routing, and multiple access points."
image: "https://wh1t35had0w.github.io/wh1t3-5hvd0w/images/mikrotik-dual-uplink-pcc-architecture.png"
---

This is the complete runbook for a MikroTik dual-uplink network using **PCC (Per Connection Classifier)**, automatic failover, HotSpot authentication, NAT and multiple wireless access points.

The configuration documented here was built and tested on a **MikroTik L009UiGS running RouterOS 7.24 stable**. The same design can be adapted to a much larger deployment by changing the interfaces, WAN addresses, HotSpot subnet, bandwidth ratio and number of downstream access points.

The important part is not the exact hardware. The important part is the order in which the network is built and the relationship between **WAN connectivity → routing → PCC → HotSpot → NAT → clients**.

![MikroTik Dual-Uplink PCC Network Architecture](https://wh1t35had0w.github.io/wh1t3-5hvd0w/images/mikrotik-dual-uplink-pcc-architecture.png)

## What This Architecture Does

The design has two independent Internet uplinks connected to a central MikroTik router.

- **Uplink 1** enters through `ether1`.
- **Uplink 2** enters through `ether2`.
- The MikroTik performs **PCC load balancing**.
- A **2:1 PCC ratio** is used in the reference deployment because Uplink 1 has approximately twice the bandwidth of Uplink 2.
- The MikroTik provides **automatic failover** when an uplink becomes unavailable.
- HotSpot clients are authenticated by the MikroTik.
- Authenticated traffic is allowed to pass through the PCC policy-routing system.
- NAT provides Internet access to the HotSpot network.
- `ether3` feeds a distribution switch and multiple access points.
- The access points provide the actual Wi-Fi coverage to users.

The resulting traffic path is:

```text
Internet / Uplink 1
        │
     Tenda F3
        │
     ether1
        │
        ├──────────────┐
        │              │
        ▼              │
   MikroTik L009       │
        ▲              │
        │              │
     ether2            │
        │              │
  Tenda TX2 Pro        │
        │              │
Internet / Uplink 2    │
                       │
             PCC + Policy Routing
                       │
                       ▼
                HotSpot Network
                       │
                    ether3
                       │
                Distribution Switch
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
        AP 1         AP 2        AP 3 ... AP 8
          │            │            │
          └────────────┴────────────┘
                       │
                 Client Devices
```

---

## Reference Deployment

The following values are from the working reference deployment. For another site, treat them as variables rather than values that must be copied blindly.

| Component | Reference value |
|---|---|
| Router | MikroTik L009UiGS |
| RouterOS | 7.24 stable |
| Uplink 1 | `ether1` |
| Uplink 1 address | `192.168.100.194/24` via DHCP |
| Uplink 1 gateway | `192.168.100.1` |
| Uplink 1 bandwidth | ~40 Mbps |
| Uplink 2 | `ether2` |
| Uplink 2 address | `192.168.101.7/24` via DHCP |
| Uplink 2 gateway | `192.168.101.1` |
| Uplink 2 bandwidth | ~20 Mbps |
| HotSpot bridge | `hotspot-bridge` |
| HotSpot gateway | `192.168.180.1` |
| HotSpot network | `192.168.180.0/22` |
| LAN bridge | `bridge` |
| LAN gateway | `192.168.88.1/24` |
| Downstream distribution | `ether3` |
| PCC ratio | 2:1 |
| Uplink 1 policy table | `TO-UPLINK1` |
| Uplink 2 policy table | `TO-UPLINK2` |
| Uplink 1 health probe | `1.1.1.1` |
| Uplink 2 health probe | `8.8.8.8` |

### Hardware Example

The reference architecture uses:

- **Tenda F3** as the device feeding **Uplink 1**.
- **Tenda TX2 Pro Wi-Fi 6** as the device feeding **Uplink 2**.
- **MikroTik L009UiGS** as the central router.
- A downstream distribution switch where required.
- Multiple access points for Wi-Fi coverage.

The Tenda devices are only examples of upstream equipment. The same MikroTik configuration can be used when the upstream connections come from different routers, ONTs, modems or other Ethernet handoffs.

---

# Part 1 — Plan the Network Before Touching the Router

The most important lesson from this deployment is to define the roles first.

Do not start by adding random mangle rules and then try to make the rest of the network fit around them.

Define these five layers first:

```text
1. WAN interfaces
2. Health-check routes
3. Policy routing tables
4. PCC connection marking
5. HotSpot + NAT integration
```

For a larger deployment, also document:

- Number of access points
- Expected number of concurrent users
- Expected bandwidth per user
- DHCP scope
- HotSpot address pool
- Authentication method
- RADIUS requirements
- Management network
- Monitoring requirements
- Backup and rollback procedure

---

# Part 2 — Prepare the MikroTik

## Step 1 — Confirm RouterOS Version

Start by checking the RouterOS version:

```routeros
/system resource print
/system package print
```

The reference router was running RouterOS 7.24 stable.

Before making production changes, export the current configuration and create a binary backup.

```routeros
/export file=BEFORE-DUAL-UPLINK-PCC
/system backup save name=BEFORE-DUAL-UPLINK-PCC
```

Do not skip the backup.

A configuration that works on one router should never be treated as a reason to remove the ability to roll back.

---

# Part 3 — Identify the Interfaces

Before configuring PCC, identify exactly which ports are connected to each uplink.

```routeros
/interface print
/interface ethernet print
```

In the reference deployment:

```text
ether1 = Uplink 1
ether2 = Uplink 2
ether3 = downstream distribution
```

The naming convention used throughout this documentation is intentionally provider-neutral:

```text
Uplink 1
Uplink 2
```

This makes the configuration easier to reuse at another site.

---

# Part 4 — Configure Uplink 1 and Uplink 2

The upstream devices provide DHCP addresses to the MikroTik.

## Uplink 1

Uplink 1 is the primary/default WAN in the reference configuration.

```routeros
/ip dhcp-client add \
    interface=ether1 \
    add-default-route=yes \
    default-route-distance=1 \
    use-peer-dns=yes \
    comment="Uplink 1 DHCP"
```

The resulting address in the reference deployment was:

```text
192.168.100.194/24
```

with gateway:

```text
192.168.100.1
```

## Uplink 2

Uplink 2 is configured without installing its DHCP default route as the main default route. This is important because the policy-routing and failover design controls where marked traffic goes.

```routeros
/ip dhcp-client add \
    interface=ether2 \
    add-default-route=no \
    use-peer-dns=no \
    comment="Uplink 2 DHCP"
```

The reference address was:

```text
192.168.101.7/24
```

with gateway:

```text
192.168.101.1
```

### Why disable the Uplink 2 DHCP default route?

Because we do not want the router to randomly choose between DHCP-installed default routes.

Instead, the routing design explicitly decides:

```text
Normal traffic       → main routing / Uplink 1
PCC Uplink 1 traffic → TO-UPLINK1
PCC Uplink 2 traffic → TO-UPLINK2
Failover             → backup route
```

---

# Part 5 — Put the WAN Interfaces in a WAN Interface List

Interface lists make firewall and NAT rules much easier to maintain.

```routeros
/interface list add name=WAN comment="Internet uplinks"
/interface list member add list=WAN interface=ether1 comment="Uplink 1"
/interface list member add list=WAN interface=ether2 comment="Uplink 2"
```

Verify:

```routeros
/interface list member print where list=WAN
```

Expected result:

```text
WAN → ether1
WAN → ether2
```

---

# Part 6 — Test Each Uplink Before PCC

Do not continue until both uplinks work independently.

Check the DHCP clients:

```routeros
/ip dhcp-client print detail
```

Check the addresses:

```routeros
/ip address print
```

Check the routes:

```routeros
/ip route print detail
```

Then test the upstream gateways.

```routeros
/ping 192.168.100.1 count=5
/ping 192.168.101.1 count=5
```

Then test the Internet through the normal routing path:

```routeros
/ping 1.1.1.1 count=10
/ping 8.8.8.8 count=10
/ping google.com count=5
```

The reference deployment produced successful Internet tests on both paths.

### Important

If an uplink cannot independently reach the Internet, PCC will not fix it.

PCC distributes connections. It does not repair a broken upstream connection.

---

# Part 7 — Create the PCC Bypass Address List

Some destinations should not be pushed through the PCC policy-routing logic.

The reference deployment used a `PCC-BYPASS` address list containing internal networks and service networks such as:

```text
10.8.0.0/20
12.0.0.0/16
13.0.0.0/16
192.168.88.0/24
192.168.100.0/24
192.168.101.0/24
192.168.180.0/22
```

The exact list must be adapted to the target environment.

Example structure:

```routeros
/ip firewall address-list
add list=PCC-BYPASS address=192.168.88.0/24 comment="LAN"
add list=PCC-BYPASS address=192.168.100.0/24 comment="Uplink 1 subnet"
add list=PCC-BYPASS address=192.168.101.0/24 comment="Uplink 2 subnet"
add list=PCC-BYPASS address=192.168.180.0/22 comment="HotSpot network"
```

If VPN networks or management networks exist, add them deliberately.

Do not copy unrelated private networks from this example into another deployment unless they actually exist there.

---

# Part 8 — Create Policy Routing Tables

RouterOS 7 uses routing tables for policy routing.

Create two tables:

```routeros
/routing table
add fib name=TO-UPLINK1 comment="Policy routing for Uplink 1"
add fib name=TO-UPLINK2 comment="Policy routing for Uplink 2"
```

Verify:

```routeros
/routing table print
```

You should see:

```text
TO-UPLINK1
TO-UPLINK2
```

---

# Part 9 — Build Recursive Health Checks

A common mistake is checking only whether the physical gateway responds.

A WAN router can remain reachable while the Internet behind it is unavailable.

For that reason, the reference design uses public probe addresses:

```text
Uplink 1 → 1.1.1.1
Uplink 2 → 8.8.8.8
```

Create the host routes through the correct upstream gateways:

```routeros
/ip route
add dst-address=1.1.1.1/32 gateway=192.168.100.1%ether1 check-gateway=ping comment="Uplink 1 health probe"
add dst-address=8.8.8.8/32 gateway=192.168.101.1%ether2 check-gateway=ping comment="Uplink 2 health probe"
```

These routes make the probe destinations reachable through the intended uplinks.

---

# Part 10 — Create the Main Failover Routes

The main routing table should have a primary path and a backup path.

The reference deployment used:

```text
Primary default path → Uplink 1
Backup default path  → Uplink 2
```

A recursive route can use the health-check destination rather than simply trusting the upstream gateway.

Example:

```routeros
/ip route
add dst-address=0.0.0.0/0 gateway=1.1.1.1 distance=1 comment="Uplink 1 primary"
add dst-address=0.0.0.0/0 gateway=8.8.8.8 distance=2 comment="Uplink 2 failover"
```

The exact recursive-route implementation should be checked with:

```routeros
/ip route print detail
```

The important result is that the primary route becomes inactive when its monitored path fails and the backup route becomes active.

---

# Part 11 — Create the Policy Routes

Each PCC routing table needs a preferred uplink and a backup uplink.

Conceptually:

```text
TO-UPLINK1
  ├── Primary → Uplink 1
  └── Backup  → Uplink 2

TO-UPLINK2
  ├── Primary → Uplink 2
  └── Backup  → Uplink 1
```

Example structure:

```routeros
/ip route
add dst-address=0.0.0.0/0 gateway=1.1.1.1 routing-table=TO-UPLINK1 distance=1 comment="TO-UPLINK1 primary"
add dst-address=0.0.0.0/0 gateway=8.8.8.8 routing-table=TO-UPLINK1 distance=2 comment="TO-UPLINK1 backup"

add dst-address=0.0.0.0/0 gateway=8.8.8.8 routing-table=TO-UPLINK2 distance=1 comment="TO-UPLINK2 primary"
add dst-address=0.0.0.0/0 gateway=1.1.1.1 routing-table=TO-UPLINK2 distance=2 comment="TO-UPLINK2 backup"
```

The purpose is connection preservation.

A connection marked for Uplink 1 should normally leave through Uplink 1. If Uplink 1 fails, the policy table must still have a usable backup path.

---

# Part 12 — Understand PCC Before Creating the Mangle Rules

PCC does not combine two Internet connections into one single TCP connection.

Instead, PCC decides which uplink should carry each connection.

For example:

```text
Client A → Connection 1 → Uplink 1
Client A → Connection 2 → Uplink 2
Client B → Connection 3 → Uplink 1
Client C → Connection 4 → Uplink 1
Client D → Connection 5 → Uplink 2
```

The reference deployment uses a **3-bucket classifier**:

```text
Bucket 0 → Uplink 1
Bucket 1 → Uplink 1
Bucket 2 → Uplink 2
```

That produces approximately:

```text
Uplink 1 = 2/3 of new connections
Uplink 2 = 1/3 of new connections
```

This is the 2:1 PCC design.

It is appropriate for a reference environment where Uplink 1 is approximately 40 Mbps and Uplink 2 is approximately 20 Mbps.

For a different bandwidth ratio, change the bucket design.

---

# Part 13 — PCC Connection Marking

The most important principle is:

> **Mark new connections, then route the entire connection using the connection mark.**

Do not repeatedly classify every packet as if it were a new connection.

The classifier used in the reference configuration is:

```text
both-addresses-and-ports:3/0
both-addresses-and-ports:3/1
both-addresses-and-ports:3/2
```

The conceptual rules are:

```routeros
/ip firewall mangle
add chain=prerouting action=mark-connection \
    new-connection-mark=UPLINK1_CONN \
    passthrough=yes \
    connection-state=new \
    dst-address-type=!local \
    hotspot=auth \
    connection-mark=no-mark \
    in-interface=hotspot-bridge \
    per-connection-classifier=both-addresses-and-ports:3/0 \
    comment="PCC 2:1 - Uplink 1 bucket 1"

add chain=prerouting action=mark-connection \
    new-connection-mark=UPLINK1_CONN \
    passthrough=yes \
    connection-state=new \
    dst-address-type=!local \
    hotspot=auth \
    connection-mark=no-mark \
    in-interface=hotspot-bridge \
    per-connection-classifier=both-addresses-and-ports:3/1 \
    comment="PCC 2:1 - Uplink 1 bucket 2"

add chain=prerouting action=mark-connection \
    new-connection-mark=UPLINK2_CONN \
    passthrough=yes \
    connection-state=new \
    dst-address-type=!local \
    hotspot=auth \
    connection-mark=no-mark \
    in-interface=hotspot-bridge \
    per-connection-classifier=both-addresses-and-ports:3/2 \
    comment="PCC 2:1 - Uplink 2 bucket"
```

### Why `connection-state=new` matters

The connection should be classified when it is created.

Once the connection receives a connection mark, subsequent packets should follow that connection's routing decision.

This prevents the same connection from bouncing between uplinks.

---

# Part 14 — Mark the Routing Table

After connection marking, convert the connection mark into a routing mark.

```routeros
/ip firewall mangle
add chain=prerouting action=mark-routing \
    new-routing-mark=TO-UPLINK1 \
    passthrough=no \
    connection-mark=UPLINK1_CONN \
    in-interface=hotspot-bridge \
    comment="Route marked connections via Uplink 1"

add chain=prerouting action=mark-routing \
    new-routing-mark=TO-UPLINK2 \
    passthrough=no \
    connection-mark=UPLINK2_CONN \
    in-interface=hotspot-bridge \
    comment="Route marked connections via Uplink 2"
```

`passthrough=no` is useful here because once the packet has received its routing decision, there is normally no reason for it to continue through unrelated mangle rules in that chain.

---

# Part 15 — PPPoE and Other Special Traffic

If PPPoE subscribers, VPNs or management networks share the same router, they should not automatically be treated like HotSpot client traffic.

The reference configuration uses a `PPPOE` interface list and a `PCC-BYPASS` address list.

Example:

```routeros
/interface list add name=PPPOE comment="PPPoE subscriber interfaces"
```

The PPP profile can dynamically add subscriber interfaces to the list when sessions come up and remove them when sessions go down.

Example concept:

```routeros
/ppp profile set [find name="BRONZE"] \
    on-up="/interface list member add list=PPPOE interface=\$interface"

/ppp profile set [find name="BRONZE"] \
    on-down="/interface list member remove [find list=PPPOE interface=\$interface]"
```

The exact PPP profiles and pools depend on the deployment.

The important design principle is to keep subscriber traffic separate from HotSpot PCC rules when the two services require different routing behavior.

---

# Part 16 — HotSpot Integration

This is where the deployment became more interesting.

A normal PCC configuration can work correctly while HotSpot clients still report:

```text
Connected to Wi-Fi
HotSpot login works
Internet does not work
```

This can happen because HotSpot adds its own dynamic firewall and NAT processing.

HotSpot creates rules for things such as:

- DNS redirection
- HTTP redirection
- Authentication
- Unauthorized clients
- Authorized clients
- Captive portal traffic
- HotSpot input/output processing

Therefore, PCC must be integrated with the HotSpot processing path instead of treating HotSpot as an ordinary LAN interface.

---

# Part 17 — Configure the HotSpot Network

The reference HotSpot gateway is:

```text
192.168.180.1
```

The network is:

```text
192.168.180.0/22
```

This provides addresses across:

```text
192.168.180.0
through
192.168.183.255
```

The HotSpot interface is:

```text
hotspot-bridge
```

Verify:

```routeros
/ip address print where interface=hotspot-bridge
/ip hotspot print detail
/ip pool print
```

---

# Part 18 — HotSpot Local Gateway Bypass

One of the troubleshooting discoveries in the reference deployment was that the HotSpot gateway itself must not be incorrectly pushed into PCC policy routing.

The bypass rule is:

```routeros
/ip firewall mangle add \
    chain=prerouting \
    action=accept \
    hotspot=auth \
    dst-address-type=local \
    comment="HOTSPOT - bypass PCC for local gateway"
```

### Rule order matters

This rule must appear **before the PCC connection-marking rules**.

In the tested configuration it was moved to position `3` in the prerouting chain.

Check the order:

```routeros
/ip firewall mangle print stats
```

You should see the local gateway bypass before the PCC marking rules.

This prevents traffic destined for the router itself, such as the HotSpot gateway, from being incorrectly treated as Internet-bound PCC traffic.

---

# Part 19 — The HotSpot `pre-hotspot` Fix

The most important HotSpot/PCC fix in the reference deployment was an explicit accept rule in the HotSpot `pre-hotspot` chain:

```routeros
/ip firewall nat add \
    chain=pre-hotspot \
    action=accept \
    hotspot=auth \
    dst-address-type=!local \
    comment="PCC-HOTSPOT-AUTH-BYPASS"
```

This was added after testing showed that the router itself had healthy Internet connectivity while authenticated HotSpot clients were still experiencing Internet problems.

The rule allows authenticated, non-local HotSpot traffic to continue through the normal routing/NAT path instead of being trapped in the HotSpot-specific processing path in a way that interferes with the PCC design.

### Important

This is not a universal rule that should be copied into every MikroTik without testing.

It was the successful integration fix for this RouterOS 7.24 HotSpot/PCC deployment.

When deploying it on another site:

1. Back up the router.
2. Confirm the HotSpot chains exist.
3. Add the rule.
4. Test authenticated clients.
5. Watch the rule counter.
6. Confirm NAT and connection tracking.

Verify it with:

```routeros
/ip firewall nat print stats where comment="PCC-HOTSPOT-AUTH-BYPASS"
```

If the counter increases while authenticated clients browse the Internet, the rule is being used.

---

# Part 20 — NAT and Masquerade

The WAN-facing masquerade rule should cover both uplinks.

The clean general approach is:

```routeros
/ip firewall nat add \
    chain=srcnat \
    action=masquerade \
    out-interface-list=WAN \
    ipsec-policy=out,none \
    comment="Masquerade Internet traffic"
```

This means traffic leaving through either member of the WAN list can be translated.

The reference deployment also contained explicit masquerade rules for several internal subnets because of the existing environment.

When creating a new deployment, do not blindly reproduce those additional subnet-specific rules. First determine whether the general WAN masquerade already covers the required traffic.

---

# Part 21 — Firewall Considerations

The firewall must protect the router without blocking legitimate HotSpot or NAT traffic.

A key WAN protection rule in the reference environment was:

```routeros
/ip firewall filter add \
    chain=forward \
    action=drop \
    connection-state=new \
    connection-nat-state=!dstnat \
    in-interface-list=WAN \
    comment="Drop unsolicited WAN traffic"
```

The rule prevents new unsolicited connections arriving from the Internet unless they are part of an intentional destination-NAT flow.

Do not remove established/related rules, HotSpot dynamic rules or service-specific exceptions simply because the PCC configuration is being changed.

---

# Part 22 — Do Not Enable FastTrack Blindly

FastTrack can interfere with policy-routing and mangle-based traffic processing depending on the design.

The reference deployment intentionally did not rely on FastTrack for this PCC/HotSpot configuration.

Before enabling FastTrack in a similar deployment, test:

- PCC connection marks
- Routing marks
- HotSpot authentication
- NAT
- Failover
- Queues
- VPN traffic
- PPPoE traffic

If policy routing is central to the architecture, leave FastTrack disabled until you have validated the complete traffic path.

---

# Part 23 — Verify the HotSpot Dynamic Rules

Once HotSpot is enabled, RouterOS creates dynamic firewall and NAT rules.

Check them with:

```routeros
/ip firewall nat print
/ip firewall filter print
```

You should see HotSpot processing such as:

```text
dstnat jump hotspot
hotspot → pre-hotspot
DNS redirects
HotSpot authentication handling
hs-unauth
hs-auth
```

The exact rule numbers can change.

Do not document rule numbers as permanent identifiers. Use comments, chain names and conditions instead.

---

# Part 24 — Test the Router Before Testing Clients

Always separate **router health** from **client health**.

Start with the router:

```routeros
/ping 1.1.1.1 count=10
/ping 8.8.8.8 count=10
/ping google.com count=5
```

The reference verification produced:

```text
1.1.1.1  → 10/10 replies
8.8.8.8  → 10/10 replies
google.com → 5/5 replies
```

This proved that the router itself had working Internet connectivity and DNS resolution.

That distinction is important.

If the router can reach the Internet but HotSpot clients cannot, stop troubleshooting the physical WAN connection and inspect HotSpot, mangle, NAT and firewall processing.

---

# Part 25 — Verify PCC Counters

PCC counters are one of the fastest ways to determine whether the classifier is actually receiving traffic.

Run:

```routeros
/ip firewall mangle print stats
```

Look for increasing packet counters on:

```text
PCC 2:1 - Uplink 1 bucket 1
PCC 2:1 - Uplink 1 bucket 2
PCC 2:1 - Uplink 2 bucket
```

The reference deployment showed active counters on all three PCC buckets.

That means new connections were being distributed across both uplinks.

---

# Part 26 — Verify Routing-Mark Counters

Next check the route-marking rules:

```routeros
/ip firewall mangle print stats
```

Look for:

```text
Route marked connections via Uplink 1
Route marked connections via Uplink 2
```

The packet counters should increase as users generate traffic.

A useful troubleshooting distinction is:

```text
PCC counters increase
        ↓
Connection marking works

Routing-mark counters increase
        ↓
Policy routing is being applied

NAT counters increase
        ↓
Traffic is being translated

Connection tracking shows replies
        ↓
End-to-end traffic is returning
```

---

# Part 27 — Verify Connection Tracking

For HotSpot clients, inspect active connections:

```routeros
/ip firewall connection print where src-address~"192.168.180."
```

You should see connections from HotSpot client addresses going to external destinations.

Examples of healthy signs include states such as:

```text
established
S
SA
```

The exact flags vary by protocol and connection state.

The important point is that you should see real external destinations and returning traffic rather than only client-to-router connections.

---

# Part 28 — Test an Actual HotSpot Client

Now test from a real phone or laptop.

### Test 1 — Connect to Wi-Fi

Confirm the client receives an address from the HotSpot network.

For example:

```text
192.168.180.x
```

### Test 2 — Confirm Gateway

The client should use:

```text
192.168.180.1
```

### Test 3 — Open the HotSpot Login

Verify the captive portal appears.

### Test 4 — Authenticate

Log in normally.

### Test 5 — Browse

Open several different websites.

### Test 6 — Generate Multiple Connections

Use normal browsing, video, DNS requests and other traffic so that PCC has multiple connections to classify.

### Test 7 — Watch the Router

At the same time, run:

```routeros
/ip firewall mangle print stats
/ip firewall nat print stats
```

The counters should increase.

---

# Part 29 — The HotSpot Disabled vs Enabled Test

This is one of the most useful troubleshooting techniques in this deployment.

If clients work when HotSpot is disabled but fail when HotSpot is enabled, the problem is probably not the physical uplink.

Use this isolation method:

```text
HotSpot disabled
        ↓
Client Internet works?
        │
       YES
        ↓
Enable HotSpot
        ↓
Client Internet fails?
        │
       YES
        ↓
Inspect HotSpot + PCC + NAT interaction
```

Do not immediately replace the PCC configuration.

First verify:

1. HotSpot dynamic NAT rules.
2. HotSpot filter rules.
3. PCC rule order.
4. Local gateway bypass.
5. `pre-hotspot` authenticated bypass.
6. NAT counters.
7. Connection tracking.

---

# Part 30 — Troubleshooting Decision Tree

## Problem: No Internet for everyone

Check:

```routeros
/interface ethernet print
/ip dhcp-client print detail
/ip address print
/ip route print detail
```

Then:

```routeros
/ping 192.168.100.1 count=5
/ping 192.168.101.1 count=5
/ping 1.1.1.1 count=5
/ping 8.8.8.8 count=5
```

If both uplinks fail, investigate the upstream devices or cabling.

If only one fails, investigate that uplink independently.

---

## Problem: Router has Internet, clients do not

Check:

```routeros
/ip hotspot active print
/ip hotspot host print
/ip firewall nat print stats
/ip firewall mangle print stats
```

Then inspect the HotSpot-specific rules.

---

## Problem: HotSpot login works but authenticated users have no Internet

Check the two HotSpot/PCC integration points first:

```text
1. HOTSPOT - bypass PCC for local gateway
2. PCC-HOTSPOT-AUTH-BYPASS
```

Verify the mangle rule is above PCC marking rules.

Verify the NAT rule exists in `pre-hotspot`.

Then check its counter:

```routeros
/ip firewall nat print stats where comment="PCC-HOTSPOT-AUTH-BYPASS"
```

---

## Problem: Only one uplink receives traffic

Check PCC counters:

```routeros
/ip firewall mangle print stats
```

If one or more classifier buckets remain at zero:

- Confirm the rule is in `prerouting`.
- Confirm `connection-state=new`.
- Confirm `connection-mark=no-mark`.
- Confirm `in-interface=hotspot-bridge` for HotSpot traffic.
- Confirm the traffic is not in `PCC-BYPASS`.
- Confirm the rule order.

---

## Problem: PCC works but websites randomly fail

Investigate asymmetric routing and connection stickiness.

A single connection should normally remain associated with one uplink.

Check:

```routeros
/ip firewall connection print
/ip firewall mangle print stats
/ip route print detail
```

Also verify that both policy tables have working routes.

---

## Problem: HotSpot gateway does not respond correctly

Check the local gateway bypass:

```routeros
/ip firewall mangle print stats
```

Look for:

```text
HOTSPOT - bypass PCC for local gateway
```

It should be before the PCC connection-marking rules.

---

## Problem: NAT appears correct but clients still cannot browse

Check the connection table:

```routeros
/ip firewall connection print where src-address~"192.168.180."
```

If connections are created but no replies return, inspect:

- Policy routing
- WAN reachability
- NAT
- Return path
- Connection marks
- Firewall drops

---

# Part 31 — Failover Testing

Failover should be tested deliberately before calling the deployment production-ready.

Because remote management may depend on an independent connection, test failover from a safe management path where possible.

## Test Uplink 1 Failure

Disconnect Uplink 1 or disable the corresponding WAN interface in a controlled maintenance window.

Then check:

```routeros
/ip route print detail
```

The Uplink 1 primary route should become inactive and the Uplink 2 route should take over.

Test:

```routeros
/ping 8.8.8.8 count=10
/ping google.com count=5
```

Then restore Uplink 1.

## Test Uplink 2 Failure

Repeat the process for Uplink 2.

The Uplink 1 path should continue carrying traffic.

### Important

Do not test failover only by unplugging the Ethernet cable.

A real deployment can experience:

```text
Cable connected + upstream Internet broken
Gateway reachable + Internet broken
DNS unavailable
Packet loss
Upstream router failure
```

This is why recursive health checking is useful.

---

# Part 32 — Test PCC After Failover

After restoring both uplinks, verify that PCC resumes normal distribution.

```routeros
/ip firewall mangle print stats
```

Generate new client traffic and watch the three PCC buckets.

Existing connections may remain on their original path. Do not interpret that as a PCC failure.

PCC is primarily concerned with assigning **new connections**.

---

# Part 33 — DNS Verification

Check RouterOS DNS:

```routeros
/ip dns print
```

The reference router was configured to accept DNS requests for the HotSpot environment and had upstream DNS servers available.

Test from the router:

```routeros
/ping google.com count=5
```

A successful hostname ping demonstrates that the router can resolve the hostname and reach the destination.

Do not assume that a single `/resolve` command with a specific server proves the entire client DNS path is broken or working. Test the actual client path as well.

---

# Part 34 — Access Point Design

The access points are downstream of the MikroTik.

The MikroTik remains the central routing and HotSpot device.

The access points should normally be treated as access-layer devices rather than additional Internet routers.

The desired logical path is:

```text
MikroTik
   │
Distribution Switch
   │
   ├── AP 1
   ├── AP 2
   ├── AP 3
   ├── AP 4
   ├── AP 5
   ├── AP 6
   ├── AP 7
   └── AP 8
```

For a larger deployment, expand the switch and AP count without changing the central PCC design.

The key requirement is that client traffic eventually reaches the HotSpot bridge/interface where the MikroTik can authenticate and route it.

---

# Part 35 — Scaling This Design

The reference configuration was built on a small router and eight access points, but the architecture is intended to be reusable.

For a larger site, scale each layer independently.

### More Internet bandwidth

Change the PCC ratio.

For example:

```text
40 Mbps + 20 Mbps → approximately 2:1
100 Mbps + 50 Mbps → approximately 2:1
200 Mbps + 100 Mbps → approximately 2:1
```

The ratio should reflect actual usable bandwidth rather than advertised package speed alone.

### More access points

Add more switch ports or additional downstream switches.

### More users

Review:

- Router CPU
- Router RAM
- Connection tracking count
- HotSpot active users
- DHCP leases
- Queue configuration
- Firewall rule count
- NAT connections
- Wireless capacity
- Switch uplink capacity

### Larger HotSpot subnet

A `/22` was used in the reference deployment.

For a larger site, select the subnet based on the expected number of clients and network design rather than copying `/22` automatically.

---

# Part 36 — Monitoring the Deployment

At minimum, monitor:

```text
Uplink 1 status
Uplink 2 status
Internet health probes
CPU
Memory
Interface traffic
PCC counters
HotSpot active users
DHCP leases
Connection count
NAT counters
Firewall drops
```

Useful commands include:

```routeros
/system resource print
/interface monitor-traffic ether1 once
/interface monitor-traffic ether2 once
/ip hotspot active print
/ip dhcp-server lease print
/ip firewall connection print count-only
/ip firewall mangle print stats
/ip firewall nat print stats
/ip route print detail
```

For a production environment, these values should eventually feed a monitoring or observability platform rather than relying exclusively on manual CLI checks.

---

# Part 37 — Backup Strategy

Do not rely on one backup.

Keep at least:

```text
BEFORE-DUAL-UPLINK-PCC
AFTER-DUAL-UPLINK-PCC
AFTER-HOTSPOT-PCC-FIX
FINAL-VALIDATED-CONFIG
```

Create both export and binary backups when appropriate:

```routeros
/export file=FINAL-VALIDATED-CONFIG
/system backup save name=FINAL-VALIDATED-CONFIG
```

The text export is useful for documentation and migration.

The binary backup is useful for restoring the router configuration on compatible hardware and RouterOS conditions.

---

# Part 38 — Production Change Checklist

Before declaring the configuration complete:

```text
[ ] Backup created
[ ] Uplink 1 reachable
[ ] Uplink 2 reachable
[ ] Internet reachable through both paths
[ ] DNS working
[ ] Routing tables created
[ ] Health probes active
[ ] Main failover tested
[ ] PCC buckets receiving traffic
[ ] Routing marks receiving traffic
[ ] HotSpot enabled
[ ] HotSpot login working
[ ] HotSpot gateway reachable
[ ] Local gateway PCC bypass in correct position
[ ] pre-hotspot authenticated bypass present
[ ] NAT counters increasing
[ ] Client Internet working
[ ] Multiple clients tested
[ ] Uplink 1 failure tested
[ ] Uplink 2 failure tested
[ ] Both uplinks restored
[ ] PCC distribution verified again
[ ] Final backup created
```

Only after all of these checks should the deployment be treated as validated.

---

# Part 39 — A Reusable Deployment Workflow

For the next large deployment, I would use this exact order:

```text
01. Document the topology
02. Identify Uplink 1 and Uplink 2
03. Back up the router
04. Configure WAN DHCP clients
05. Verify both gateways
06. Verify independent Internet access
07. Create WAN interface list
08. Create PCC-BYPASS list
09. Create policy routing tables
10. Create recursive health probes
11. Create main failover routes
12. Create policy-table primary/backup routes
13. Configure PCC connection marking
14. Configure routing marks
15. Configure HotSpot
16. Add HotSpot local gateway bypass
17. Add pre-hotspot authenticated bypass
18. Configure NAT
19. Verify firewall
20. Test router Internet
21. Test HotSpot login
22. Test authenticated client Internet
23. Verify PCC counters
24. Verify NAT counters
25. Verify connection tracking
26. Test Uplink 1 failure
27. Test Uplink 2 failure
28. Restore both links
29. Verify PCC again
30. Export final configuration
31. Create final backup
32. Record the final architecture
```

The order matters because every stage gives you a known-good checkpoint before the next layer is introduced.

---

# Part 40 — Read-Only Health Check Script

For future deployments, I use a separate verification script instead of immediately changing configuration.

The following is intentionally read-only:

```routeros
:put "==============================================="
:put "DUAL-UPLINK PCC + HOTSPOT HEALTH CHECK"
:put "==============================================="

:put ""
:put "--- SYSTEM ---"
/system resource print
/system package print

:put ""
:put "--- INTERFACES ---"
/interface print

:put ""
:put "--- WAN DHCP ---"
/ip dhcp-client print detail

:put ""
:put "--- ADDRESSES ---"
/ip address print

:put ""
:put "--- ROUTES ---"
/ip route print detail

:put ""
:put "--- ROUTING TABLES ---"
/routing table print

:put ""
:put "--- HOTSPOT ---"
/ip hotspot print detail
/ip hotspot active print
/ip hotspot host print

:put ""
:put "--- PCC / MANGLE COUNTERS ---"
/ip firewall mangle print stats

:put ""
:put "--- NAT COUNTERS ---"
/ip firewall nat print stats

:put ""
:put "--- CONNECTION COUNT ---"
/ip firewall connection print count-only

:put ""
:put "--- INTERNET TESTS ---"
/ping 1.1.1.1 count=5
/ping 8.8.8.8 count=5
/ping google.com count=5

:put ""
:put "==============================================="
:put "HEALTH CHECK COMPLETE"
:put "==============================================="
```

This script deliberately does not modify the router.

That makes it suitable for first-pass troubleshooting.

---

# Part 41 — Do Not Build a Blind One-Click Production Script

It is tempting to create one script that immediately creates every rule.

For a large production site, that can be dangerous.

Interface names, gateway addresses, HotSpot subnets, existing firewall rules, VPN networks, PPPoE services and queue policies can all be different.

A safer automation model is:

```text
DISCOVER
   ↓
VALIDATE
   ↓
BACKUP
   ↓
PLAN
   ↓
APPLY
   ↓
VERIFY
   ↓
BACKUP AGAIN
```

The automation should stop if a prerequisite is missing instead of assuming that the environment matches the reference deployment.

---

# Part 42 — Recommended Variables for a Deployment Generator

For a future automated installer, define the following variables at the top:

```text
UPLINK1_INTERFACE=ether1
UPLINK1_GATEWAY=192.168.100.1
UPLINK1_PROBE=1.1.1.1
UPLINK1_BANDWIDTH=40M

UPLINK2_INTERFACE=ether2
UPLINK2_GATEWAY=192.168.101.1
UPLINK2_PROBE=8.8.8.8
UPLINK2_BANDWIDTH=20M

HOTSPOT_INTERFACE=hotspot-bridge
HOTSPOT_GATEWAY=192.168.180.1
HOTSPOT_NETWORK=192.168.180.0/22

UPLINK1_TABLE=TO-UPLINK1
UPLINK2_TABLE=TO-UPLINK2

PCC_RATIO=2:1
```

The generator can then create the RouterOS configuration from those variables.

This is much safer than hard-coding one particular site into a script intended for a different large deployment.

---

# Part 43 — Known Issues Observed During the Reference Deployment

## Ethernet/PPPoE Link Flapping

The reference router also showed repeated link up/down events on the interface carrying a PPPoE subscriber connection.

That should be treated as a separate physical or downstream-device issue rather than automatically blaming PCC.

Investigate:

- Ethernet cable
- Connector
- Switch port
- ONT
- Power
- Duplex/speed negotiation
- Subscriber device

Useful commands:

```routeros
/interface ethernet monitor ether5 once
/log print where message~"ether5"
```

A WAN load-balancing problem and a physically unstable downstream Ethernet link are two different troubleshooting problems.

---

# Part 44 — What I Learned From the Troubleshooting

The biggest lesson was that a network can look healthy at one layer and still fail at another.

For example:

```text
WAN links healthy
      ↓
Router Internet healthy
      ↓
PCC counters increasing
      ↓
BUT
      ↓
HotSpot users cannot browse
```

That does not automatically mean the WAN configuration is wrong.

It means the traffic path must be traced layer by layer.

The useful troubleshooting model is:

```text
Physical
   ↓
DHCP
   ↓
Gateway
   ↓
Internet
   ↓
Routing
   ↓
PCC
   ↓
HotSpot
   ↓
NAT
   ↓
Connection tracking
   ↓
Client
```

Do not jump over layers.

---

# Part 45 — Final Architecture

The completed architecture looks like this:

```text
                    INTERNET
                 /             \
                /               \
        UPLINK 1                 UPLINK 2
        Tenda F3              Tenda TX2 Pro
            │                       │
          ether1                  ether2
            │                       │
            └──────────┬────────────┘
                       │
                       ▼
              ┌─────────────────┐
              │    MikroTik     │
              │                 │
              │ PCC 2:1         │
              │ Policy Routing  │
              │ Failover        │
              │ HotSpot         │
              │ Firewall        │
              │ NAT             │
              └────────┬────────┘
                       │
                    ether3
                       │
                       ▼
              ┌─────────────────┐
              │ Distribution    │
              │ Switch          │
              └───────┬─────────┘
                      /|\\
                     / | \\
                    /  |  \\
                   ▼   ▼   ▼
                  APs  APs  APs
                   │   │   │
                   └───┴───┘
                       │
                       ▼
                CLIENT DEVICES
```

The MikroTik is the control point.

The upstream routers provide the Internet connections.

The distribution switch expands the LAN.

The access points provide wireless coverage.

PCC decides which new connections use which uplink.

Recursive health checks determine whether an uplink is actually usable.

HotSpot handles user access.

NAT provides Internet translation.

---

# Final Verification Commands

Before handing the network over, I use this short verification set:

```routeros
/system resource print
/interface print
/ip dhcp-client print detail
/ip address print
/ip route print detail
/routing table print
/ip hotspot print detail
/ip hotspot active print
/ip firewall mangle print stats
/ip firewall nat print stats
/ip firewall connection print count-only
```

Then:

```routeros
/ping 1.1.1.1 count=10
/ping 8.8.8.8 count=10
/ping google.com count=5
```

Then test from at least two real HotSpot clients while watching:

```routeros
/ip firewall mangle print stats
/ip firewall nat print stats
/ip firewall connection print where src-address~"192.168.180."
```

Finally create the validated backup:

```routeros
/export file=FINAL-DUAL-UPLINK-PCC-HOTSPOT-VALIDATED
/system backup save name=FINAL-DUAL-UPLINK-PCC-HOTSPOT-VALIDATED
```

---

# Conclusion

This deployment is more than simply connecting two Internet routers to a MikroTik.

The stable architecture comes from combining several independent mechanisms correctly:

```text
Two WAN links
      +
Recursive health checks
      +
Policy routing
      +
PCC connection marking
      +
HotSpot integration
      +
Authenticated pre-hotspot bypass
      +
NAT
      +
Firewall
      +
Downstream access points
      =
Reusable dual-uplink network architecture
```

The most important principle for the next large deployment is simple:

> **Build, verify and document each layer before moving to the next one.**

That makes troubleshooting dramatically easier and turns a configuration that was manually engineered once into a repeatable deployment pattern.
