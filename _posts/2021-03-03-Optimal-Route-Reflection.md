---
layout: post
title: BGP Optimal Route Reflection
--- 

iBGP Route Reflection is an important technique used by many iBGP-enabled networks. By relaxing the full-mesh requirement of iBGP sessions and using designated Route Reflectors per cluster, we no longer have to configure a full-mesh of neighbor statements on every PE router. However, Route Reflectors *by default* will not only reflect routes between clients, but will also select a best-path as would any other router running BGP. The best-path selected by the RR could often not be the same best-path that would've been selected by a client when IGP metric is considered in the path selection process. 

---

## Topology 
<a href="/images/bgp-orr.png" target="_blank"> <img src="/images/bgp-orr.png"/></a>

The RR only sees it's point of view IGP-wise, not the view of the RR clients, by default. In our example, we have AS3 advertising 33.33.33.33/32 toward both AS1 and AS2. AS1 peers with AS64510 at R1, and AS2 peers with AS64510 at R8. On RR1 we have set IGP metric higher on the interface toward R8, and lower on the interface toward R1. And vice-versa for RR2, lower toward R8 and higher toward R1. 

You may already see where I'm going with this, I want RR1 to select R1's 33.33.33.33/32 advertisement for egress and I want RR2 to select R8's. Here's the output of "show route 33.33.33.33" on RR1 and RR2. 

```
root@RR1> show route 33.33.33.33 detail | no-more

inet.0: 31 destinations, 32 routes (31 active, 0 holddown, 0 hidden)
33.33.33.33/32 (2 entries, 1 announced)
        *BGP    Preference: 170/-101
                Next hop type: Indirect, Next hop index: 0
                Address: 0xc388914
                Next-hop reference count: 2
                Source: 1.1.1.1
                Next hop type: Router, Next hop index: 594
                Next hop: 100.64.0.29 via ge-0/0/3.0, selected
                Label operation: Push 299776
                Label TTL action: prop-ttl
                Load balance label: Label 299776: None;
                Label element ptr: 0xd7cbc48
                Label parent element ptr: 0x0
                Label element references: 1
                Label element child references: 0
                Label element lsp id: 0
                Session Id: 0x141
                Protocol next hop: 1.1.1.1
                Indirect next hop: 0xc2c0704 1048575 INH Session ID: 0x143
                State: <Active Int Ext>
                Local AS: 64510 Peer AS: 64510
                Age: 5:15       Metric2: 65554
                Validation State: unverified
                Task: BGP_64510.1.1.1.1
                Announcement bits (3): 0-KRT 4-BGP_RT_Background 5-Resolve tree 4
                AS path: 1 3 I
                Accepted
                Localpref: 100
                Router ID: 1.1.1.1
         BGP    Preference: 170/-101
                Next hop type: Indirect, Next hop index: 0
                Address: 0xc3889dc
                Next-hop reference count: 1
                Source: 8.8.8.8
                Next hop type: Router, Next hop index: 587
                Next hop: 100.64.0.27 via ge-0/0/2.0, selected
                Label element ptr: 0xd7cb860
                Label parent element ptr: 0x0
                Label element references: 1
                Label element child references: 0
                Label element lsp id: 0
                Session Id: 0x140
                Protocol next hop: 8.8.8.8
                Indirect next hop: 0xc2c0884 1048574 INH Session ID: 0x142
                State: <Int Ext>
                Inactive reason: IGP metric
                Local AS: 64510 Peer AS: 64510
                Age: 5:15       Metric2: 65555
                Validation State: unverified
                Task: BGP_64510.8.8.8.8
                AS path: 2 3 I
                Accepted
                Localpref: 100
                Router ID: 8.8.8.8
```

```
root@RR2> show route 33.33.33.33 detail | no-more

inet.0: 31 destinations, 32 routes (31 active, 0 holddown, 0 hidden)
33.33.33.33/32 (2 entries, 1 announced)
        *BGP    Preference: 170/-101
                Next hop type: Indirect, Next hop index: 0
                Address: 0xc388c34
                Next-hop reference count: 2
                Source: 8.8.8.8
                Next hop type: Router, Next hop index: 587
                Next hop: 100.64.0.33 via ge-0/0/3.0, selected
                Label operation: Push 299776
                Label TTL action: prop-ttl
                Load balance label: Label 299776: None;
                Label element ptr: 0xd7c6e78
                Label parent element ptr: 0x0
                Label element references: 1
                Label element child references: 0
                Label element lsp id: 0
                Session Id: 0x141
                Protocol next hop: 8.8.8.8
                Indirect next hop: 0xc2c0404 1048574 INH Session ID: 0x142
                State: <Active Int Ext>
                Local AS: 64510 Peer AS: 64510
                Age: 6:52       Metric2: 65544
                Validation State: unverified
                Task: BGP_64510.8.8.8.8
                Announcement bits (3): 0-KRT 4-BGP_RT_Background 5-Resolve tree 4
                AS path: 2 3 I
                Accepted
                Localpref: 100
                Router ID: 8.8.8.8
         BGP    Preference: 170/-101
                Next hop type: Indirect, Next hop index: 0
                Address: 0xc3886bc
                Next-hop reference count: 1
                Source: 1.1.1.1
                Next hop type: Router, Next hop index: 588
                Next hop: 100.64.0.31 via ge-0/0/2.0, selected
                Label operation: Push 299824
                Label TTL action: prop-ttl
                Load balance label: Label 299824: None;
                Label element ptr: 0xd7c6568
                Label parent element ptr: 0x0
                Label element references: 1
                Label element child references: 0
                Label element lsp id: 0
                Session Id: 0x140
                Protocol next hop: 1.1.1.1
                Indirect next hop: 0xc2bfc84 1048575 INH Session ID: 0x143
                State: <Int Ext>
                Inactive reason: IGP metric
                Local AS: 64510 Peer AS: 64510
                Age: 6:52       Metric2: 65545
                Validation State: unverified
                Task: BGP_64510.1.1.1.1
                AS path: 1 3 I
                Accepted
                Localpref: 100
                Router ID: 1.1.1.1
```

Indeed, we see that RR1 selects active route toward R1 and RR2 selects R8 as egress. In detailed output, we see a reason for the non-active routes not being chosen under <span style="color:orange">Inactive reason: IGP Metric"</span>

---
*NOTE*

You see the RR selects the active route due to IGP cost/metric. You can see the full JunOS BGP path selection process <a href="https://www.juniper.net/documentation/en_US/junos/topics/reference/general/routing-protocols-address-representation.html" target="_blank">here</a>

---

## What happens when a client only has a session to one route reflector? 

Let's introduce the problem. When a RR client only receives routes from 1 RR, the results can be undesirable.

On R7 RR client, I will deactivate the session to RR2. This will mimic an iBGP session failure of some sort whether that be human-error or otherwise. 

```
root@R7# deactivate protocols bgp group ibgp neighbor 22.22.22.22
```

Now, show route on R7 to 33.33.33.33
```
root@R7> show route 33.33.33.33 detail

inet.0: 33 destinations, 33 routes (33 active, 0 holddown, 0 hidden)
33.33.33.33/32 (1 entry, 1 announced)
        *BGP    Preference: 170/-101
                Next hop type: Indirect, Next hop index: 0
                Address: 0xc389724
                Next-hop reference count: 2
                Source: 11.11.11.11
                Next hop type: Router, Next hop index: 0
                Next hop: 100.64.0.13 via ge-0/0/1.0 weight 0x1, selected
                Label-switched-path to-r5
                Label operation: Push 299824, Push 299776(top)
                Label TTL action: prop-ttl, prop-ttl(top)
                Load balance label: Label 299824: None; Label 299776: None;
                Label element ptr: 0xd916850
                Label parent element ptr: 0xd9163f0
                Label element references: 2
                Label element child references: 0
                Label element lsp id: 2
                Session Id: 0x0
                Next hop: 100.64.0.20 via ge-0/0/2.0 weight 0x1
                Label-switched-path to-r2
                Label operation: Push 299776, Push 299776(top)
                Label TTL action: prop-ttl, prop-ttl(top)
                Load balance label: Label 299776: None; Label 299776: None;
                Label element ptr: 0xd917098
                Label parent element ptr: 0xd916738
                Label element references: 2
                Label element child references: 0
                Label element lsp id: 4
                Session Id: 0x0
                Protocol next hop: 1.1.1.1
```

The selected route is AS path 1 3, with a next-hop of 1.1.1.1 which is the R1 egress router. This is undesirable, and if we weren't running MPLS could even cause <span style="color:red">routing loops.</span> Imagine a world where R7 forwards this 33.33.33.33/32 bound traffic toward R6, and R6 has selected the AS path 2 3 instead of 1 3, with egress router R8 instead of R1. That spawns a loop between R6 and R7 for 33.33.33.33/32 destined traffic. Thank you MPLS for doing what you do best, creating the forwarding abstraction between ingress and egress routers. We owe you one. 

So, what steps could we take to keep this from happening? We need to alter the default behavior of RR's only advertising the best route from the RR point of view. Let's go over some options. 

### 1. BGP Add-Path 
Add Path is a BGP extension that allows speakers to send and receive multiple paths for the same prefix. Add Path is simple to configure, like so on RR1 and R7 (Remember that RR2 to R7 iBGP is still deactivated).

```
root@R7# show | compare
[edit protocols bgp group ibgp]
+     family inet {
+         unicast {
+             add-path {
+                 receive;
+             }
+         }
+     }

root@RR1# show | compare
[edit protocols bgp group ibgp]
+     family inet {
+         unicast {
+             add-path {
+                 send {
+                     path-count 2;
+                 }
+             }
+         }
+     }
```

And now we are able to receive 2 path advertisements from RR1 instead of just the best path. This yields our desired path through AS2 AS3 from R7. 

```
root@R7> show route 33.33.33.33

inet.0: 33 destinations, 34 routes (33 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

33.33.33.33/32     *[BGP/170] 00:00:08, localpref 100, from 11.11.11.11
                      AS path: 2 3 I, validation-state: unverified
                    >  to 100.64.0.10 via ge-0/0/0.0
                    [BGP/170] 00:00:08, localpref 100, from 11.11.11.11
                      AS path: 1 3 I, validation-state: unverified
                    >  to 100.64.0.13 via ge-0/0/1.0, label-switched-path to-r5
                       to 100.64.0.20 via ge-0/0/2.0, label-switched-path to-r2
```

### 2. Internet in a VRF 
When you configure a VRF, best practice is to make your VRF prefixes globally unique within your AS with a route-distinguisher format like LoopbackIP:Number. If you use AS:Number, you risk 2 PE's advertising the same prefix, and again only one being chosen as "best" by the route reflector because it sees them as the exact same AS:Number:Prefix combination. So if you do "Internet in a VRF" correctly, the route distinguisher makes every route advertisement globally unique. 

This one requires a little more configuration overhead in a brownfield network as you can imgagine. I need to move internet from inet.0 to internet.inet.0 in a L3VPN. I won't show it all, but just know that we are creating internet L3VPN on every PE router, and RR's advertise prefixes from bgp.l3vpn.0. 

After configuring Internet VRF/L3VPN on R1, R8, R7 and family inet-vpn unicast on all involved routers. 

```
root@RR1# run show route table bgp.l3vpn.0

bgp.l3vpn.0: 4 destinations, 4 routes (4 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

1.1.1.1:123:33.33.33.33/32
                   *[BGP/170] 00:00:00, localpref 100, from 1.1.1.1
                      AS path: 1 3 I, validation-state: unverified
                    >  to 100.64.0.29 via ge-0/0/3.0, Push 300032, Push 299776(top)
1.1.1.1:123:192.168.0.0/31
                   *[BGP/170] 00:00:01, localpref 100, from 1.1.1.1
                      AS path: I, validation-state: unverified
                    >  to 100.64.0.29 via ge-0/0/3.0, Push 300032, Push 299776(top)
8.8.8.8:123:10.0.0.0/31
                   *[BGP/170] 00:11:07, localpref 100, from 8.8.8.8
                      AS path: I, validation-state: unverified
                    >  to 100.64.0.27 via ge-0/0/2.0, Push 299888
8.8.8.8:123:33.33.33.33/32
                   *[BGP/170] 00:11:06, localpref 100, from 8.8.8.8
                      AS path: 2 3 I, validation-state: unverified
                    >  to 100.64.0.27 via ge-0/0/2.0, Push 299888
```
Notice the two unique routes for 33.33.33.33/32 

```
root@R7# run show route 33.33.33.33 detail

internet.inet.0: 3 destinations, 4 routes (3 active, 0 holddown, 0 hidden)
33.33.33.33/32 (2 entries, 1 announced)
        *BGP    Preference: 170/-101
                Route Distinguisher: 8.8.8.8:123
                Next hop type: Indirect, Next hop index: 0
                Address: 0xc389724
                Next-hop reference count: 6
                Source: 11.11.11.11
                Next hop type: Router, Next hop index: 622
                Next hop: 100.64.0.10 via ge-0/0/0.0, selected
                Label operation: Push 299888
                Label TTL action: prop-ttl
                Load balance label: Label 299888: None;
                Label element ptr: 0xc71f5d0
                Label parent element ptr: 0xd916300
                Label element references: 1
                Label element child references: 0
                Label element lsp id: 0
                Session Id: 0x141
                Protocol next hop: 8.8.8.8
                Label operation: Push 299888
                Label TTL action: prop-ttl
                Load balance label: Label 299888: None;
                Indirect next hop: 0xc2bef04 1048574 INH Session ID: 0x14c
                State: <Secondary Active Int Ext ProtectionCand>
                Local AS: 64510 Peer AS: 64510
                Age: 1:30       Metric2: 10
                Validation State: unverified
                Task: BGP_64510.11.11.11.11
                Announcement bits (1): 0-KRT
                AS path: 2 3 I  (Originator)
                Cluster list:  11.11.11.11
                Originator ID: 8.8.8.8
                Communities: target:123:123
                Import Accepted
                VPN Label: 299888
                Localpref: 100
                Router ID: 11.11.11.11
                Primary Routing Table bgp.l3vpn.0
         BGP    Preference: 170/-101
                Route Distinguisher: 1.1.1.1:123
                Next hop type: Indirect, Next hop index: 0
                Address: 0xc3897ec
                Next-hop reference count: 5
                Source: 11.11.11.11
                Next hop type: Router, Next hop index: 0
                Next hop: 100.64.0.13 via ge-0/0/1.0 weight 0x1, selected
                Label-switched-path to-r5
                Label operation: Push 300656, Push 299824, Push 299776(top)
                Label TTL action: prop-ttl, prop-ttl, prop-ttl(top)
                Load balance label: Label 300656: None; Label 299824: None; Label 299776: None;
                Label element ptr: 0xd917cf0
                Label parent element ptr: 0xd916850
                Label element references: 2
                Label element child references: 0
                Label element lsp id: 0
                Session Id: 0x0
                Next hop: 100.64.0.20 via ge-0/0/2.0 weight 0x1
                Label-switched-path to-r2
                Label operation: Push 300656, Push 299776, Push 299776(top)
                Label TTL action: prop-ttl, prop-ttl, prop-ttl(top)
                Load balance label: Label 300656: None; Label 299776: None; Label 299776: None;
                Label element ptr: 0xc71f558
                Label parent element ptr: 0xd917098
                Label element references: 2
                Label element child references: 0
                Label element lsp id: 0
                Session Id: 0x0
                Protocol next hop: 1.1.1.1
                Label operation: Push 300656
                Label TTL action: prop-ttl
                Load balance label: Label 300656: None;
                Indirect next hop: 0xc2bf804 1048576 INH Session ID: 0x15f
                State: <Secondary Int Ext Changed ProtectionCand>
                Inactive reason: IGP metric
                Local AS: 64510 Peer AS: 64510
                Age: 1  Metric2: 30
                Validation State: unverified
                Task: BGP_64510.11.11.11.11
                AS path: 1 3 I  (Originator)
                Cluster list:  11.11.11.11
                Originator ID: 1.1.1.1
                Communities: target:123:123
                Import Accepted
                VPN Label: 300656
                Localpref: 100
                Router ID: 11.11.11.11
                Primary Routing Table bgp.l3vpn.0
```
And again, R7 can now see and decide between the two egress points. 

### 3. BGP Optimal Route Reflection 

BGP-ORR is a simple solution to the problem of suboptimal route reflection. Instead of an RR advertising a route based on its best-path calculation from its point of view, the RR can use a client's point of view to select a route for advertisement to those clients. BGP-ORR uses the information from the IGP link-state database to calculate paths from client to advertising routers. 

In implementation, you are expected to configure a group of peers at the RR, configure ORR for that group, and select a primary and/or backup client for that group that will be used to calculate IGP metric. In other words, IGP metric is not calculated from every single client in the group to each egress router, rather it is calculated from a selected client for that group to the egress routers. Upon reading the BGP-ORR daft (found in references bottom of this post), it was recommended to configure a primary and backup node so you aren't relying on one specific client for BGP-ORR to operate. Think of regional POPs, configuring a couple different routers in the region as these designated BGP-ORR clients. 

In my example, I used R7's loopback as the igp-primary node because it's the node we are concerned with. 
```
root@RR1# show | compare
[edit protocols bgp group ibgp]
+     optimal-route-reflection {
+         igp-primary 7.7.7.7;
+     }
```

And I'm able to see the calculations done from R7 to different nodes using BGP-ORR
```
root@RR1# run show isis bgp-orr
BGP ORR Peer Group: ibgp
  Primary: 7.7.7.7, active
IPv4/IPv6 ORR Routes
--------------------
Prefix                  L Version   Metric Type
1.1.1.1/32              2      64       30 int
2.2.2.2/32              2      64       20 int
3.3.3.3/32              2      64       10 int
4.4.4.4/32              2      64       20 int
5.5.5.5/32              2      64       20 int
6.6.6.6/32              2      64       10 int
7.7.7.7/32              2      64        0 int
8.8.8.8/32              2      64       10 int
22.22.22.22/32          2      64       10 int
100.64.0.0/31           2      64       30 int
100.64.0.2/31           2      64       20 int
100.64.0.4/31           2      64       20 int
100.64.0.6/31           2      64       20 int
100.64.0.8/31           2      64       20 int
100.64.0.10/31          2      64       10 int
100.64.0.12/31          2      64       10 int
100.64.0.14/31          2      64       20 int
100.64.0.16/31          2      64       30 int
100.64.0.18/31          2      64       20 int
100.64.0.20/31          2      64       10 int
100.64.0.22/31          2      64       20 int
100.64.0.24/31          2      64       20 int
100.64.0.30/31          2      64       30 int
100.64.0.32/31          2      64       10 int
```

RR1 is advertising the optimized route to R7
```
root@RR1# run show route advertising-protocol bgp 7.7.7.7 detail

inet.0: 30 destinations, 31 routes (30 active, 0 holddown, 0 hidden)
  33.33.33.33/32 (2 entries, 2 announced)
 BGP group ibgp type Internal
     Nexthop: 8.8.8.8
     Localpref: 100
     AS path: [64510] 2 3 I
     Cluster ID: 11.11.11.11
     Originator ID: 8.8.8.8
```

And there is connectivity!
```
root@R7> ping 33.33.33.33 source 7.7.7.7
PING 33.33.33.33 (33.33.33.33): 56 data bytes
64 bytes from 33.33.33.33: icmp_seq=0 ttl=62 time=9.548 ms
64 bytes from 33.33.33.33: icmp_seq=1 ttl=62 time=8.514 ms
^C
--- 33.33.33.33 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max/stddev = 8.514/9.031/9.548/0.517 ms

root@R7> show route 33.33.33.33 detail

inet.0: 33 destinations, 33 routes (33 active, 0 holddown, 0 hidden)
33.33.33.33/32 (1 entry, 1 announced)
        *BGP    Preference: 170/-101
                Next hop type: Indirect, Next hop index: 0
                Address: 0xc389788
                Next-hop reference count: 2
                Source: 11.11.11.11
                Next hop type: Router, Next hop index: 602
                Next hop: 100.64.0.10 via ge-0/0/0.0, selected
                Label element ptr: 0xd916300
                Label parent element ptr: 0x0
                Label element references: 1
                Label element child references: 0
                Label element lsp id: 0
                Session Id: 0x141
                Protocol next hop: 8.8.8.8
                Indirect next hop: 0xc2be904 1048576 INH Session ID: 0x16d
                State: <Active Int Ext>
                Local AS: 64510 Peer AS: 64510
                Age: 31:36      Metric2: 10
                Validation State: unverified
                Task: BGP_64510.11.11.11.11
                Announcement bits (2): 0-KRT 5-Resolve tree 4
                AS path: 2 3 I  (Originator)
                Cluster list:  11.11.11.11
                Originator ID: 8.8.8.8
                Accepted
                Localpref: 100
                Router ID: 11.11.11.11
```

---
## AN ISSUE I HAD
I had some significant issues when inet.3 was populated by LDP. The advertisements would not be that of the BGP-ORR calculated route. Instead, the advertisements would be from the RR's point of view again. My thought is that there is no mapping between IS-IS route next hop calculated in BGP-ORR and the active (preferred) LDP next hop route in inet.3. I'd imagine this is something that could easily be fixed in the future, being as BGP-ORR would provide very little value to SP's running L3VPN with a ASN:# naming conventioned for Route Distinguishers without the added inet.3 mapping functionality. <span style="color:blue">YES, you can workaround this by copying IS-IS routes into inet.3 with rib-group, or by changing resolution rib for bgp.l3vpn.0.. But I hardly consider that a fix.</span>

I started a conversation about this on <a href="https://www.reddit.com/r/Juniper/comments/lwymt8/bgporr_on_vmx/?utm_source=share&utm_medium=web2x&context=3" target="_blank">reddit</a> where a user helped me realize the inet.3 issue that I didn't expect right away. I assumed the IGP-LDP mapping would be there for these BGP-ORR calculations. 

---

### Configs
Configs found <a href="/configs/bgp-orr.txt">here</a>

### References 

<a href="https://datatracker.ietf.org/doc/draft-ietf-idr-bgp-optimal-route-reflection/" target="_blank">https://datatracker.ietf.org/doc/draft-ietf-idr-bgp-optimal-route-reflection/</a>

<a href="https://tools.ietf.org/html/rfc4456">https://tools.ietf.org/html/rfc4456</a>

<a href="https://www.noction.com/blog/bgp-optimal-route-reflection-alternative-to-bgp-add-path">https://www.noction.com/blog/bgp-optimal-route-reflection-alternative-to-bgp-add-path</a>

<a href="https://www.juniper.net/documentation/en_US/junos/topics/topic-map/bgp-rr.html#id-understanding-bgp-optimal-route-reflection">https://www.juniper.net/documentation/en_US/junos/topics/topic-map/bgp-rr.html#id-understanding-bgp-optimal-route-reflection</a>

