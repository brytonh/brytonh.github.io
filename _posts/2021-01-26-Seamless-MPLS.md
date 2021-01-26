---
layout: post
title: Seamless MPLS
--- 

Seamless MPLS is directly derived from Interprovider VPN Option C, as BGP Labeled Unicast is used between administrative domains. In Interprovider VPN Option C, eBGP-LU is used to extend LSPs between two Autonomous Systems at the AS border. In Seamless MPLS, BGP-LU is used to extend the MPLS network and LSPs beyond regional administrative boundaries. The boundary can be an Autonomous System border like with Interprovider VPN Option C mentioned, but it could also be an IGP area or sub-AS. I'd like to focus on a topology with an ISP that has geographically separate regions, and the need for MPLS LSPs end-to-end. The ISP is using IS-IS throughout their whole network, but their regions are Level 1 domains and the core is their Level 2 backbone.  

### Some remarks before we get started..
* I'm in no way saying Seamless MPLS is the "way to go" or popular, in fact maybe the opposite <a href="https://blog.ipspace.net/2020/08/worth-reading-seamless-suffering.html" target="_blank">*cough* </a>
* Label stacking for MPLS transport paths isn't new, reference LDP over RSVP

---

## Topology: 
![](/images/SeamlessMPLS.png)

## Device roles 
1. Access - Routers used to hand off L3 or L2 services directly to access switches or to customers directly 
2. BNG - Broadband Network Gateway, Services head-end for the ISP, where pseudowires would terminate for L3 service from the Access node per region 
3. Border - Regional border router facing core, functioning as our iBGP-LU RRs
4. Core - Nodes doing label switching within our Level 2 core, transport packets between regions 

## Starting point
* IS-IS is configured according to the topology, Level 2 in the core 49.0001 and Level 1 in the regional areas. 
* All of the intra-area LDP LSPs in the regions are configured 
* The intra-area bidirectional LSPs are configured from Core-1 to Core-3
* All necessary address families for interfaces are configured already (MPLS, ISO, INET)

---

##iBGP-LU Configuration
So, instead of having to signal an LDP LSP (with L2->L1 leaking or longest-match configured) or running RSVP with some inter-domain extension (expand loose hop) or bypassing TED, we can extend LSPs using BGP-LU. When you use eBGP-LU, this is pretty straight forward as with Option C. eBGP-LU sets the next-hop self toward the eBGP-LU peer, and now the eBGP peers are label pitstops on a packet's journey from source to destination. 

However, we aren't using eBGP-LU. We don't have separate AS's in our topology. How do we make iBGP-LU behave similarly to inter-AS option C? The answer: <span style="color:blue"> Route Reflection</span>

With the borders acting as RR's, we also configured them to set next-hop self for advertised iBGP-LU routes. This will guarantee that for traffic region->core and core->region will be forwarded by means of iBGP-LU LSP. The RR's will have a BGP peering to one another dedicated to sharing regional routes. 

Border-1 Config: 
```
root@Border-1# show protocols bgp | no-more
group ibgp-west {
    type internal;
    local-address 3.3.3.3;
    family inet {
        labeled-unicast {
            rib {
                inet.3;
            }
        }
    }
    export nhs;
    cluster 3.3.3.3;
    neighbor 2.2.2.2;
    neighbor 1.1.1.1;
}
group ibgp-core {
    type internal;
    local-address 3.3.3.3;
    family inet {
        labeled-unicast {
            rib {
                inet.3;
            }
        }
    }
    export nhs;
    neighbor 7.7.7.7;
}
```

Border-2: 
```
root@Border-2# show protocols bgp | no-more
group ibgp-east {
    type internal;
    local-address 7.7.7.7;
    family inet {
        labeled-unicast {
            rib {
                inet.3;
            }
        }
    }
    export nhs;
    cluster 7.7.7.7;
    neighbor 8.8.8.8;
    neighbor 9.9.9.9;
}
group ibgp-core {
    type internal;
    local-address 7.7.7.7;
    family inet {
        labeled-unicast {
            rib {
                inet.3;
            }
        }
    }
    export nhs;
    neighbor 3.3.3.3;
}
```

*Core routers need not run BGP, as they are only LSRs, label switching only* 

---

We have iBGP-LU configured at the borders and at the core. We've configured our border routers to next-hop self and act as RR for labeled prefixes advertised from inet.3. Let's originate prefixes from the Access nodes into iBGP-LU. 

Access-1:
```
root@Access-1# show | compare
[edit]
+  policy-options {
+      policy-statement int-only {
+          from interface lo0.0;
+          then accept;
+      }
+  }
[edit routing-options]
+   interface-routes {
+       rib-group inet interfaces;
+   }
+   rib-groups {
+       interfaces {
+           import-rib [ inet.0 inet.3 ];
+           import-policy int-only;
+       }
+   }
[edit protocols bgp group ibgp]
+    export int-only;

root@Access-1# show protocols bgp
group ibgp {
    type internal;
    local-address 1.1.1.1;
    family inet {
        labeled-unicast {
            rib {
                inet.3;
            }
        }
    }
    export int-only;
    neighbor 2.2.2.2;
    neighbor 3.3.3.3;
}

root@Access-1# run show route advertising-protocol bgp 3.3.3.3 detail
inet.3: 5 destinations, 5 routes (5 active, 0 holddown, 0 hidden)
* 1.1.1.1/32 (1 entry, 1 announced)
 BGP group ibgp type Internal
     Route Label: 3
     Nexthop: Self
     Flags: Nexthop Change
     Localpref: 100
     AS path: [64510] I
     Entropy label capable
```

Access-2: 
```
root@Access-2# show | compare
[edit]
+  policy-options {
+      policy-statement int-only {
+          from interface lo0.0;
+          then accept;
+      }
+  }
[edit routing-options]
+   interface-routes {
+       rib-group inet interfaces;
+   }
+   rib-groups {
+       interfaces {
+           import-rib [ inet.0 inet.3 ];
+           import-policy int-only;
+       }
+   }
[edit protocols bgp group ibgp]
+    export int-only;

root@Access-2# show protocols bgp
group ibgp {
    type internal;
    local-address 9.9.9.9;
    family inet {
        labeled-unicast {
            rib {
                inet.3;
            }
        }
    }
    export int-only;
    neighbor 7.7.7.7;
    neighbor 8.8.8.8;
}

root@Access-2# run show route advertising-protocol bgp 7.7.7.7 detail
inet.3: 5 destinations, 5 routes (5 active, 0 holddown, 0 hidden)
* 9.9.9.9/32 (1 entry, 1 announced)
 BGP group ibgp type Internal
     Route Label: 3
     Nexthop: Self
     Flags: Nexthop Change
     Localpref: 100
     AS path: [64510] I
     Entropy label capable
```

We are advertising the loopbacks from the Access nodes into iBGP-LU. Let's see what the Border-1 does with Access-1's 1.1.1.1/32 prefix upon receiving it. Remember, it's configured as a RR. 

```
root@Border-1> show route receive-protocol bgp 1.1.1.1 detail
inet.3: 2 destinations, 3 routes (2 active, 0 holddown, 0 hidden)
  1.1.1.1/32 (2 entries, 2 announced)
     Accepted
     Route Label: 3
     Nexthop: 1.1.1.1
     Localpref: 100
     AS path: I

root@Border-1> show route advertising-protocol bgp 7.7.7.7 detail
inet.3: 2 destinations, 3 routes (2 active, 0 holddown, 0 hidden)
  1.1.1.1/32 (2 entries, 2 announced)
 BGP group ibgp-core type Internal
     Route Label: 299808
     Nexthop: Self
     Flags: Nexthop Change
     Localpref: 100
     AS path: [64510] I
     Cluster ID: 3.3.3.3
     Originator ID: 1.1.1.1
     Entropy label capable
```

Border-1 advertises to Border-2 a label of 299808 to reach 1.1.1.1/32. What does Border-1 do when it receives this label? Let's check the LFIB mpls.0 table. 
```
root@Border-1> show route label 299808
mpls.0: 10 destinations, 10 routes (10 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

299808             *[VPN/170] 00:06:24, metric2 1, from 1.1.1.1
                    >  to 100.64.0.2 via ge-0/0/0.0, Swap 299776
```

It swaps 299808 for a label of 299776 and sents the packet toward West-region BNG. Where did this 299776 label come from? Well, remember we are running LDP for LSPs intra-region. 
```
root@Border-1> show route table inet.3 1.1.1.1/32

inet.3: 2 destinations, 3 routes (2 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both
1.1.1.1/32         *[LDP/9] 10:33:02, metric 1
                    >  to 100.64.0.2 via ge-0/0/0.0, Push 299776
                    [BGP/170] 00:15:34, localpref 100, from 1.1.1.1
                      AS path: I, validation-state: unverified
                    >  to 100.64.0.2 via ge-0/0/0.0, Push 299776
```

There it is, we are stitching the incoming iBGP-LU terminated LSP to the active forwarding LSP provided to us by LDP. It's true that we advertised 1.1.1.1/32 from Access-1 via iBGP-LU, but the active forwarding LSP is via LDP.

So Border-1 is advertising 299808 label for prefix 1.1.1.1/32 to Border-2, let's see what Border-2 does with it. 
```
root@Border-2# run show route receive-protocol bgp 3.3.3.3 detail
inet.3: 4 destinations, 5 routes (4 active, 0 holddown, 0 hidden)
* 1.1.1.1/32 (1 entry, 1 announced)
     Accepted
     Route Label: 299808
     Nexthop: 3.3.3.3
     Localpref: 100
     AS path: I  (Originator)
     Cluster list:  3.3.3.3
     Originator ID: 1.1.1.1
     Entropy label capable, next hop field matches route next hop

root@Border-2# run show route advertising-protocol bgp 9.9.9.9 detail

inet.3: 4 destinations, 5 routes (4 active, 0 holddown, 0 hidden)
* 1.1.1.1/32 (1 entry, 1 announced)
 BGP group ibgp-east type Internal
     Route Label: 299808
     Nexthop: Self
     Flags: Nexthop Change
     Localpref: 100
     AS path: [64510] I  (Originator)
     Cluster list:  3.3.3.3
     Originator ID: 1.1.1.1
     Cluster ID: 7.7.7.7
     Entropy label capable
```

Oh would you look at that, we advertised label 299808 for the prefix 1.1.1.1/32 to Access-2. This is a good time to mention that MPLS labels are only *locally significant* for forwarding. 

What's Border-2 do when it receives label 299808, checking LFIB? 
```
[edit]
root@Border-2# run show route table mpls.0 label 299808 detail

mpls.0: 11 destinations, 11 routes (11 active, 0 holddown, 0 hidden)
299808 (1 entry, 1 announced)
        *VPN    Preference: 170
                Next hop type: Indirect, Next hop index: 0
                Address: 0xc388aa4
                Next-hop reference count: 2
                Source: 3.3.3.3
                Next hop type: Router, Next hop index: 588
                Next hop: 100.64.0.10 via ge-0/0/1.0, selected
                Label-switched-path to-border1
                Label operation: Swap 299808, Push 299776(top)
                Label TTL action: prop-ttl, prop-ttl(top)
                Load balance label: Label 299808: None; Label 299776: None;
```

So we swap label 299808 for 299808 (label received for iBGP-LU prefix 1.1.1.1/32) but we also pushed on 299776, why? 

```
root@Border-2# run show route 1.1.1.1/32 protocol bgp detail

inet.0: 20 destinations, 20 routes (20 active, 0 holddown, 0 hidden)

inet.3: 4 destinations, 5 routes (4 active, 0 holddown, 0 hidden)
1.1.1.1/32 (1 entry, 1 announced)
        *BGP    Preference: 170/-101
                Next hop type: Indirect, Next hop index: 0
                Address: 0xc3884c8
                Next-hop reference count: 2
                Source: 3.3.3.3
                Next hop type: Router, Next hop index: 0
                Next hop: 100.64.0.10 via ge-0/0/1.0, selected
                Label-switched-path to-border1
                Label operation: Push 299808, Push 299776(top)
                Label TTL action: prop-ttl, prop-ttl(top)
                Load balance label: Label 299808: None; Label 299776: None;
                Label element ptr: 0xd97c530
                Label parent element ptr: 0xd97c648
                Label element references: 1
                Label element child references: 0
                Label element lsp id: 0
                Session Id: 0x0
                Protocol next hop: 3.3.3.3
                Label operation: Push 299808

root@Border-2# run show route table inet.3 3.3.3.3 detail

inet.3: 4 destinations, 5 routes (4 active, 0 holddown, 0 hidden)
3.3.3.3/32 (1 entry, 1 announced)
        State: <FlashAll>
        *RSVP   Preference: 7/1
                Next hop type: Router, Next hop index: 0
                Address: 0xc388914
                Next-hop reference count: 3
                Next hop: 100.64.0.10 via ge-0/0/1.0, selected
                Label-switched-path to-border1
                Label operation: Push 299776
```

The anwser lies in our received iBGP-LU route from the other border. Yes, we received 299808 as the BGP-LU label, but we can't actually reach Border-1 from Border-2 "directly." Instead we resolve the BGP protocol next hop in inet.3, and find the RSVP LSP toward Border-1 that gives us the reachability we needed. That's where the "push 299776" comes from. It's our transport label between borders. 

Finally, we check what Access-2 sees for prefix 1.1.1.1/32
```
root@Access-2# run show route receive-protocol bgp 7.7.7.7 detail
inet.3: 6 destinations, 6 routes (6 active, 0 holddown, 0 hidden)
* 1.1.1.1/32 (1 entry, 1 announced)
     Accepted
     Route Label: 299808
     Nexthop: 7.7.7.7
     Localpref: 100
     AS path: I  (Originator)
     Cluster list:  7.7.7.7 3.3.3.3
     Originator ID: 1.1.1.1

root@Access-2# run show route table inet.3 1.1.1.1

inet.3: 6 destinations, 6 routes (6 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

1.1.1.1/32         *[BGP/170] 00:08:36, localpref 100, from 7.7.7.7
                      AS path: I, validation-state: unverified
                    >  to 100.64.0.14 via ge-0/0/2.0, Push 299808, Push 299792(top)
```

There it is, the iBGP-LU route that'll get us to Access-1 from Access-2. The top label will get us to the BGP protocol next hop *Border-2* and the bottom label represents the Border->Border of the regions. 

---

Now that we have this LSP end to end between Access nodes, let's put a service onto it to at least prove that it works by configuring a simple pseudowire via t-LDP. 

![](/images/l2circuit.png)

```
root@Access-1# show protocols l2circuit
neighbor 9.9.9.9 {
    interface ge-0/0/0.0 {
        virtual-circuit-id 123;
    }
}

[edit]
root@Access-1# show interfaces ge-0/0/0
encapsulation ethernet-ccc;
unit 0;
```

Access-2: 
```
root@Access-2# show protocols l2circuit
neighbor 1.1.1.1 {
    interface ge-0/0/0.0 {
        virtual-circuit-id 123;
    }
}

[edit]
root@Access-2# show interfaces ge-0/0/0
encapsulation ethernet-ccc;
unit 0;
```

```
root@Access-1# run show l2circuit connections
Layer-2 Circuit Connections:

Legend for connection status (St)
EI -- encapsulation invalid      NP -- interface h/w not present
MM -- mtu mismatch               Dn -- down
EM -- encapsulation mismatch     VC-Dn -- Virtual circuit Down
CM -- control-word mismatch      Up -- operational
VM -- vlan id mismatch           CF -- Call admission control failure
OL -- no outgoing label          IB -- TDM incompatible bitrate
NC -- intf encaps not CCC/TCC    TM -- TDM misconfiguration
BK -- Backup Connection          ST -- Standby Connection
CB -- rcvd cell-bundle size bad  SP -- Static Pseudowire
LD -- local site signaled down   RS -- remote site standby
RD -- remote site signaled down  HS -- Hot-standby Connection
XX -- unknown

Legend for interface status
Up -- operational
Dn -- down
Neighbor: 9.9.9.9
    Interface                 Type  St     Time last up          # Up trans
    ge-0/0/0.0(vc 123)        rmt   Up     Jan 26 16:13:09 2021           1
      Remote PE: 9.9.9.9, Negotiated control-word: Yes (Null)
      Incoming label: 299808, Outgoing label: 299840
      Negotiated PW status TLV: No
      Local interface: ge-0/0/0.0, Status: Up, Encapsulation: ETHERNET
      Flow Label Transmit: No, Flow Label Receive: No
```

```
root@Access-1# run show route table l2circuit.0

l2circuit.0: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

9.9.9.9:CtrlWord:5:123:Local/96
                   *[L2CKT/7] 00:08:48, metric2 1
                    >  to 100.64.0.1 via ge-0/0/2.0, Push 299872, Push 299792(top)
```

And now we run some pings through from PC to PC
```
VPCS> ping 10.0.0.2

84 bytes from 10.0.0.2 icmp_seq=1 ttl=64 time=11.306 ms
84 bytes from 10.0.0.2 icmp_seq=2 ttl=64 time=10.520 ms
84 bytes from 10.0.0.2 icmp_seq=3 ttl=64 time=10.375 ms
84 bytes from 10.0.0.2 icmp_seq=4 ttl=64 time=9.577 ms
84 bytes from 10.0.0.2 icmp_seq=5 ttl=64 time=8.365 ms
```

## Wireshark Image 
![](/images/l2circuit-wireshark.png)

299840 is the bottom L2circuit label - terminates on Access-2 
299872 is the middle iBGP-LU label - swapped at Border-1
299792 is the topmost LDP label - egress of LSP is Border-1 

We didn't really end up using the BNG nodes, the lab got a bit long. But I imagine you can get the picture of terminating L3 services via pseudowire or L2 to a BNG.

## Notes about Seamless MPLS 
* iBGP-LU is not magic, as you've seen it's just iBGP and still requires reachability so we end up with label stacks. Topmost label is always representing our most immediate LSP path 
* The end-product is end-to-end LSPs, but this lab was extremely small scale - imagine running this with thousands of access nodes. Automation helps some
* I'm not sure I would want the 3am on-call guy troubleshooting Seamless MPLS paths

## References 
<a href="https://www.juniper.net/assets/us/en/local/pdf/whitepapers/2000452-en.pdf" target="_blank">Juniper End-to-End MPLS Whitepaper</a>
<a href="https://www.juniper.net/documentation/en_US/release-independent/nce/topics/concept/nce-141-solution-overview.html" target="_blank">Juniper config example, Seamless MPLS</a>
<a href="https://www.sanog.org/resources/sanog21/SANOG21_Seamless%20MPLS_%20focus%20on%20flexible%20service%20delivery.pdf" target="_blank">SANOG Seamless MPLS deck</a>
<a href="http://ibest.ee/documentation/Juniper/T%20Series/Network%20Scaling%20with%20BGP%20Labeled%20Unicast.pdf" target="_blank">Network Scaling with BGP Labeled-Unicast</a>



