---
layout: post
title: 4 Label Stack in JunOS
--- 

Admittingly, this is going to end up feeling a lot more like a MPLS party trick than a blog post with real learning potential. But, we are going to go full nerd with this one and exceed the "normal" 3 label stack we might have been used to with something like seamless MPLS. Today, we are shooting for: VPN,BGP-LU,LDP,RSVP. We are doing L3VPN Interprovider Option C, with LDP Tunneling over RSVP (LDPoRSVP).

What's the point? Just to see how JunOS reacts to the label stack size larger than 3 by default for vMX and vSRX routers. Also - for a core leveraging LDPoRSVP or doing some flavor of seamless MPLS and adding on some interprovider VPN this is a completely realistic scenario. 

---

## Topology: 
![](/images/4labelstack.png)

## What I've already configured 

I've already configured RSVP LSP's between PE1 and P3, and LDP tunneling over these LSPs. I've also configured the IP routing in 64510 via OSPF backbone area 0 and in 64511 as ISIS single level 2 area. P4 will advertise its loopback /32 into LDP, and P3 will pass this along as an advertisement as well as its own /32 via targeted LDP session to PE1. PE1 has LDP tunneling configured for the RSVP LSP from PE1->P3, so this will be viewed as the best next hop and the LDP routes for 192.168.1.4/32 and 192.168.1.5/32 will be tossed into inet.3. 

Keep in mind LDP must always follow IGP next-hop, and you can see configuring LDP-tunneling in JunOS enables OSPF shortcuts in the background as hidden routes

```
root@PE1# run show ldp database
Input label database, 192.168.1.1:0--192.168.1.3:0
Labels received: 3
  Label     Prefix
 299792      192.168.1.1/32
      3      192.168.1.3/32
 299776      192.168.1.4/32

Output label database, 192.168.1.1:0--192.168.1.3:0
Labels advertised: 3
  Label     Prefix
      3      192.168.1.1/32
 299776      192.168.1.3/32
 299792      192.168.1.4/32

root@PE1# run show route table inet.3 hidden
inet.3: 3 destinations, 6 routes (2 active, 0 holddown, 3 hidden)
+ = Active Route, - = Last Active, * = Both

100.64.1.4/31       [OSPF] 00:10:34, metric 3
                    >  to 100.64.1.1 via ge-0/0/0.0, label-switched-path to-r3
192.168.1.3/32      [OSPF] 00:10:34, metric 2
                    >  to 100.64.1.1 via ge-0/0/0.0, label-switched-path to-r3
192.168.1.4/32      [OSPF] 00:10:34, metric 3
                    >  to 100.64.1.1 via ge-0/0/0.0, label-switched-path to-r3

root@PE1# run show route table inet.3 192.168.1.4
inet.3: 3 destinations, 6 routes (2 active, 0 holddown, 3 hidden)
+ = Active Route, - = Last Active, * = Both

192.168.1.4/32     *[LDP/9] 00:11:08, metric 1
                    >  to 100.64.1.1 via ge-0/0/0.0, label-switched-path to-r3
```

---

## What's next? 

We have an operational LDPoRSVP LSP between PE1 and P4. We also have LDP operating in AS64511. We need to configure eBGP-LU, as well as iBGP-LU sessions in both AS's. 

On P4: 
```
[edit protocols]
+   bgp {
+       family inet {
+           labeled-unicast;
+           rib {
+           inet.3;
+           }
+       }
+       group ebgp {
+           type external;
+           peer-as 64511;
+           neighbor 100.64.2.1;
+       }
+       group ibgp {
+           type internal;
+           local-address 192.168.1.4;
+           neighbor 192.168.1.1;
+       }
+   }
```

On P5: 
```
[edit protocols]
+   bgp {
+       family inet {
+           labeled-unicast;
+      		 rib {
+          		 inet.3;
+       	 } 
+       }
+       group ebgp {
+           type external;
+           peer-as 64510;
+           neighbor 100.64.2.0;
+       }
+       group ibgp {
+           type internal;
+           local-address 192.168.1.5;
+           neighbor 192.168.1.6;
+       }
+   }
```

On PE1:
```
[edit protocols]
+   bgp {
+       group ibgp {
+           type internal;
+           local-address 192.168.1.1;
+           family inet {
+               labeled-unicast;
+      		 	rib {
+           		inet.3;
+       		}
+           }
+           neighbor 192.168.1.4;
+       }
+   }
```

On PE6:
```
[edit protocols]
+   bgp {
+       group ibgp {
+           type internal;
+           local-address 192.168.1.6;
+           family inet {
+               labeled-unicast;
+       		rib {
+           		inet.3;
+       	}
+           }
+           neighbor 192.168.1.5;
+       }
+   }
```

Why did I choose "rib inet.3" under family inet labeled-unicast? Chris covers this really well in [this post about BGP-LU](https://www.networkfuntimes.com/bgp-labeled-unicast-on-juniper-routers-for-jncie-sp-students/). By keeping the BGP families inet unicast and inet labeled-unicast separate, we can learn and advertise prefixes from/to both tables without issue. Because of the likelihood of this happening in the real world, I've chosen to keep labeled advertisements in inet.3 RIB. 

My iBGP sessions are established, but I need to get the PE1 loopback into inet.3 so I can actually advertise it. Because remember, that's the RIB I'm using on every router for push/pull of labeled-unicast prefixes. I'm going to use a rib-group and interface-routes to do this. 

```
[edit routing-options]
+   interface-routes {
+       rib-group inet copy-lo0;
+   }
+   rib-groups {
+       copy-lo0 {
+           import-rib [ inet.0 inet.3 ];
+       }
+   }
```

And then we apply export policy to ibgp groups on both PE1 and PE6 to get the loopbacks into iBGP

```
[edit]
+  policy-options {
+      policy-statement export-lo0 {
+          from interface lo0.0;
+          then accept;
+      }
+  }
[edit protocols bgp group ibgp]
+    export export-lo0;


[edit]
root@PE1# run show route advertising-protocol bgp 192.168.1.4 detail

inet.3: 6 destinations, 9 routes (5 active, 0 holddown, 3 hidden)
* 192.168.1.1/32 (1 entry, 1 announced)
 BGP group ibgp type Internal
     Route Label: 3
     Nexthop: Self
     Flags: Nexthop Change
     Localpref: 100
     AS path: [64510] I
     Entropy label capable
```

P5 is receiving the route for 192.168.1.1/32 with a label in the other AS64511 
```
root@P5# run show route receive-protocol bgp 100.64.2.0 detail
inet.3: 2 destinations, 2 routes (1 active, 0 holddown, 1 hidden)
* 192.168.1.1/32 (1 entry, 1 announced)
     Accepted
     Route Label: 299824
     Nexthop: 100.64.2.0
     AS path: 64510 I Unrecognized Attributes: 11 bytes
     AS path:  Attr flags e0 code 1c: 00 01 04 04 c0 a8 01 01
```

But P5 isn't advertising this route to PE6, which is a problem
```
root@P5# ...show route advertising-protocol bgp 192.168.1.6 detail
inet.3: 2 destinations, 2 routes (1 active, 0 holddown, 1 hidden)
* 192.168.1.1/32 (1 entry, 1 announced)
 BGP group ibgp type Internal
     BGP label allocation failure: family mpls not enabled on interface
     Nexthop: Not advertised
     Flags: Nexthop Change
     Localpref: 100
     AS path: [64511] 64510 I Unrecognized Attributes: 11 bytes
     AS path:  Attr flags e0 code 1c: 00 01 04 04 c0 a8 01 01
```

And the reason given is "nexthop: not advertised".. A little cryptic. Well, the real reason is that MPLS is not enabled on the inter-AS link, and it needs to be to allow for MPLS traffic to forward on the link. 

```
[edit interfaces ge-0/0/3 unit 0]
+      family mpls;
```

```
root@P5# run show route advertising-protocol bgp 192.168.1.6 detail

inet.3: 2 destinations, 2 routes (1 active, 0 holddown, 1 hidden)
* 192.168.1.1/32 (1 entry, 1 announced)
 BGP group ibgp type Internal
     Route Label: 299776
     Nexthop: Self
     Flags: Nexthop Change
     Localpref: 100
     AS path: [64511] 64510 I Unrecognized Attributes: 11 bytes
     AS path:  Attr flags e0 code 1c: 00 01 04 04 c0 a8 01 01
 ```

PE1 has a route in inet.3 for PE6 and vice-versa 
```
root@PE1# run show route protocol bgp

inet.0: 10 destinations, 10 routes (10 active, 0 holddown, 0 hidden)

inet.3: 7 destinations, 10 routes (6 active, 0 holddown, 3 hidden)
+ = Active Route, - = Last Active, * = Both

192.168.1.6/32     *[BGP/170] 00:00:06, localpref 100, from 192.168.1.4
                      AS path: 64511 I, validation-state: unverified
                    >  to 100.64.1.1 via ge-0/0/0.0, label-switched-path to-r3
```

The Problem is that we need this route in inet.0 to establish iBGP peering between the two. Rib-group to the rescue again for this simple operation. 

```
root@PE1# show | compare
[edit routing-options rib-groups]
    copy-lo0 { ... }
+   copy-ebgplu-to-inet0 {
+       import-rib [ inet.3 inet.0 ];
+   }
[edit protocols bgp group ibgp family inet labeled-unicast]
+        rib-group copy-ebgplu-to-inet0;

root@PE1# run show route protocol bgp
inet.0: 11 destinations, 11 routes (11 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

192.168.1.6/32     *[BGP/170] 00:00:08, localpref 100, from 192.168.1.4
                      AS path: 64511 I, validation-state: unverified
                    >  to 100.64.1.1 via ge-0/0/0.0, label-switched-path to-r3

inet.3: 7 destinations, 10 routes (6 active, 0 holddown, 3 hidden)
+ = Active Route, - = Last Active, * = Both

192.168.1.6/32     *[BGP/170] 00:02:33, localpref 100, from 192.168.1.4
                      AS path: 64511 I, validation-state: unverified
                    >  to 100.64.1.1 via ge-0/0/0.0, label-switched-path to-r3
```

```
root@PE1# run traceroute 192.168.1.6 source 192.168.1.1
traceroute to 192.168.1.6 (192.168.1.6) from 192.168.1.1, 30 hops max, 52 byte packets
 1  100.64.1.1 (100.64.1.1)  11.708 ms  8.978 ms  12.350 ms
     MPLS Label=299776 CoS=0 TTL=1 S=0
     MPLS Label=299776 CoS=0 TTL=1 S=0
     MPLS Label=299856 CoS=0 TTL=1 S=1
 2  100.64.1.3 (100.64.1.3)  14.698 ms  13.704 ms  15.115 ms
     MPLS Label=299776 CoS=0 TTL=1 S=0
     MPLS Label=299856 CoS=0 TTL=2 S=1
 3  100.64.1.5 (100.64.1.5)  22.004 ms  18.510 ms  19.497 ms
     MPLS Label=299856 CoS=0 TTL=1 S=1
 4  100.64.2.1 (100.64.2.1)  28.827 ms  28.709 ms  29.727 ms
     MPLS Label=299808 CoS=0 TTL=1 S=1
 5  192.168.1.6 (192.168.1.6)  22.977 ms  24.452 ms  24.528 ms
 ```

 And connectivity between the two PE's is looking fine. 

 ---

 As you saw in that last traceroute from PE1->PE6, that was already a 3-label-push operation. The bottom label was the BGP-LU label, the middle was the LDP label for the P4 router, and the RSVP label was the top label that gives us LSP reachability to P4. So what's more label? No problem, we are almost there. 

 For simplicy, I'm using a couple direct subnets and vrf-table-label for the L3VPN. We've done enough configuration, it's time to be a little lazy to hurry and see this 4 label stack at work.

PE1:
 ```
 [edit interfaces lo0]
+    unit 1 {
+        family inet {
+            address 11.11.11.11/32;
+        }
+    }
[edit]
+  routing-instances {
+      test {
+          instance-type vrf;
+          interface lo0.1;
+          vrf-target target:1:1;
+          vrf-table-label;
+      }
+  }
[edit routing-options]
+  route-distinguisher-id 192.168.1.1;
[edit protocols bgp]
     group ibgp { ... }
+    group ebgp-vpn {
+        local-address 192.168.1.1;
+        multihop;
+        family inet-vpn {
+            unicast;
+        }
+        peer-as 64511;
+        neighbor 192.168.1.6;
+    }
```

PE6:
```
root@PE6# show | compare | no-more
[edit interfaces lo0]
+    unit 1 {
+        family inet {
+            address 66.66.66.66/32;
+        }
+    }
[edit]
+  routing-instances {
+      test {
+          instance-type vrf;
+          interface lo0.1;
+          vrf-target target:1:1;
+          vrf-table-label;
+      }
+  }
[edit routing-options]
+  route-distinguisher-id 192.168.1.6;
[edit protocols bgp]
     group ibgp { ... }
+    group ebgp-vpn {
+        local-address 192.168.1.6;
+        family inet-vpn {
+            unicast;
+        }
+        multihop;
+        peer-as 64510;
+        neighbor 192.168.1.1;
+    }
```

---

HOORAY! We did it, let's see our amazing success and go out for pizza! 

```
root@PE1# run show route table test hidden

test.inet.0: 2 destinations, 2 routes (1 active, 0 holddown, 1 hidden)
+ = Active Route, - = Last Active, * = Both

66.66.66.66/32      [BGP/170] 00:00:21, localpref 100, from 192.168.1.6
                      AS path: 64511 I, validation-state: unverified
                       Unusable
```

Ah man, that's not what we expected, let's get a little more detail. 

```
root@PE1# run show route table test hidden detail

test.inet.0: 2 destinations, 2 routes (1 active, 0 holddown, 1 hidden)
66.66.66.66/32 (1 entry, 0 announced)
         BGP    Preference: 170/-101
                Route Distinguisher: 192.168.1.6:8
               <mark> Next hop type: Unusable, Next hop index: 0</mark>
                Address: 0xc3872d0
                Next-hop reference count: 3
                State: <Secondary Hidden Ext Changed ProtectionCand>
                Local AS: 64510 Peer AS: 64511
                Age: 1:16
                Validation State: unverified
                Task: BGP_64511.192.168.1.6
                AS path: 64511 I
                Communities: target:1:1
                Import Accepted
                VPN Label: 16
                Localpref: 100
                Router ID: 192.168.1.6
                Primary Routing Table bgp.l3vpn.0
```

Hmm, there's that super informative next-hop unusable error. Maybe this is somehow related to the greater-than-3-label feat we have accomplished? 

If you check out this [link](https://www.juniper.net/documentation/en_US/junos/topics/reference/configuration-statement/maximum-labels-edit-interfaces-unit-family-mpls.html) you will blatantly see that the default stack depth is 3. Let's change this to 4 and see if we this resolves. 

```
root@PE1# show | compare
[edit interfaces ge-0/0/0 unit 0 family mpls]
+      maximum-labels 4;
```

```
root@PE1# run show route table test detail protocol bgp

test.inet.0: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)
66.66.66.66/32 (1 entry, 1 announced)
        *BGP    Preference: 170/-101
                Route Distinguisher: 192.168.1.6:8
                Next hop type: Indirect, Next hop index: 0
                Address: 0xc387dc0
                Next-hop reference count: 3
                Source: 192.168.1.6
                Next hop type: Router, Next hop index: 611
                Next hop: 100.64.1.1 via ge-0/0/0.0, selected
                Label-switched-path to-r3
             <mark>   Label operation: Push 16, Push 299856, Push 299776, Push 299824(top) </mark>
```

Beautiful! The route is there. 

```
root@PE1# run ping 66.66.66.66 routing-instance test
PING 66.66.66.66 (66.66.66.66): 56 data bytes
64 bytes from 66.66.66.66: icmp_seq=0 ttl=60 time=58.453 ms
64 bytes from 66.66.66.66: icmp_seq=1 ttl=60 time=24.834 ms
64 bytes from 66.66.66.66: icmp_seq=2 ttl=60 time=24.627 ms
```

And the pings are a great success! 

Quickly, let's follow the labels. 
1. PE1 pushes on VPN label 16, BGP-LU label 299856 for next-hop 192.168.1.4, LDP label 299776 for 192.168.1.4, and RSVP label 299824 for targeted LDP shortest next-hop 192.168.1.3. 
2. P2 receives label stack, sees top label 299824 and performs a pop operation as it is the PHP for the RSVP tunnel to P3
3. P3 receives 3-label-stack with top label 299776 and does a pop operation as it is the PHP for the LDP LSP to 192.168.1.4
4. P4 receives 2-label-stack with top label 299856, and swaps it for 299808 label that is received from P5 for prefix 192.168.1.6/32
5. P5 receives 2-label-stack with top label 299808 and pops the top label, as it is the PHP for the iBGP-LU/LDP session from P6, and P5 sends the 1-label-stack toward PE6
6. PE6 receives a 1-label-stack of just the VPN label 16, and because of vrf-table-label PE6 pops the 16 and sends bare IP packets to vrf test for lookup where it meets lo0.1 processing.

## PCAP Image
Here is the beautiful 4 label stack in wireshark as it leaves PE1 toward P2, in all its glory. 
![](/images/4labelstack-pcap.png)

**If you made it this far, thanks for following along.** 

