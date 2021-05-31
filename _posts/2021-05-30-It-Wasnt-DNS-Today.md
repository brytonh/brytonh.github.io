---
layout: post
title: Today, It wasn't DNS 
--- 

But it was MTU. 

Join me in the replay of a troubleshooting scenario. This is my first post like this, so join me and we'll see if it's any good. 

---

It's a Saturday afternoon, and an alert goes through about a specific core P router going down in the network. It's a pure P router, and there's built in redundancy so it normally is not the biggest deal. I check with the electric side of the house in our company, and they say power will be restored soon to the location. Great! 

Or so I thought. Upon further look at the spam of alerts in slack, a PE router and the access switches hanging off of it were also down. Well that's not normal, that site has a backup wireless PTP link and power is most definitely up there. I can even login to the PE router via the loopback address. What gives? 

```
bherdes@xyz> show ospf neighbor | grep .85 
100.64.1.98      ge-0/1/3.2998          Full      100.64.127.85      1    38
```

I do a "show ospf neighbor" on the PE router and of course I see the neighbor P router that's on the other side of the wireless PTP in "full" state. Okay, and of course we are running MPLS - specifically LDP at this part of the network - and I see the neighborship operational with "show ldp neighbor".

```
bherdes@xyz> show ldp neighbor | grep .85 
100.64.1.98        ge-0/1/3.2998      100.64.127.85:0          11
```

However, I do NOT have LDP routes to the PE loopback from the border edges of the network. That LDP route is most definitely not making in there. 

```
bherdes@xxyyzz> show route table inet.3 100.64.127.85/32 

inet.3: 49 destinations, 49 routes (49 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both
```

So, I login to the P router facing the PE via wireless PTP, and run a "show ldp database session 100.64.127.85"... This command is going to allow be to see the label mappings from the PE as well as my label mappings as this P router to the PE. 

```
root@2101-SHERIFF-ACX> show ldp database session 100.64.127.85 | no-more 
Input label database, 100.64.127.72:0--100.64.127.85:0

Output label database, 100.64.127.72:0--100.64.127.85:0
  Label     Prefix
 299952      100.64.127.32/32
 299968      100.64.127.35/32
 299984      100.64.127.36/32

Trimmed for brevity

```

Okay, wow that's a huge problem. From the perspective of the P router I'm receiving absolutely ZERO prefixes via LDP from PE. I check the PE with the same command, and it is definitely sending many label mappings... again what gives? 

This is when I decided I wanted to run a packet trace and dig into the LDP messages. I started the packet trace via SPAN and filtered for port 646/tcp/udp for LDP on this specific interface. (LDP will use UDP/646 for hello's and discovery, and TCP/646 for FEC label exchange and next-hop address messages) 

## Show me the Trace

<a href="/images/ldp_trace1.png" target="_blank"> <img src="/images/ldp_trace1.png"/></a>

So far so good here, we see the hello's from each end on UDP/646. We also see a TCP 3-way handshake on TCP/646 at the very end of the trace pictured here. If you're wondering what the RST is and the FIN/ACK in the trace, it's because I cleared the LDP session to start this trace and get all initial messages. 

### Drilling into LDP 

<a href="/images/ldp_initial.png" target="_blank"> <img src="/images/ldp_initial.png"/></a>

LDP Initialization message is pictured here above, and basically this is where LSR ID is shared from each side by the neighbors. They also share label space identifiers and other information. 

### LDP Address Message 

LDP follows the IGP, always. Have you ever wondered how this works? How does an MPLS-enabled router, more specifically for LDP, know that it's using the correct label mapping for the correct neighbor in comparison to the IGP routing table? The answer lies in the IGP next hop and in the LDP address message. 

<a href="/images/ldp_address.png" target="_blank"> <img src="/images/ldp_address.png"/></a>

Here is an LDP address message from one of the neighbors. It is basically sharing its available next hop IP addresses to its LDP neighbors. By doing this, an LDP-enabled router can look at the LDP mapping from a neighbor for a prefix, and check the RIB to see if the prefix is actually found with a next hop matching one of these addresses. If so, we program that in our LFIB. 

### LDP Label Mapping

 <a href="/images/ldp_mapping1.png" target="_blank"> <img src="/images/ldp_mapping1.png"/></a>

 Here you can notice this is an LDP Label Mapping packet from the P router to the PE. You may also notice that this trace compiled together 3 TCP segments into one for our viewing. In this specific scenario, I don't mind but normally I would consider disabling that automatic aggregation of segments when looking at what ends up being our specific issue. Nothing stands out about this specific message in the trace, so let's move on to the label mapping that was from the PE to the P (the one that seemed like it was not receieved.) After all, we earlier noted that it was the P not receiving the PE's label mappings appropriately, not the other way around. 

---

 <a href="/images/ldp_fail.png" target="_blank"> <img src="/images/ldp_fail.png"/></a>

Ah, here's what you probably expected to see. We see TCP retransmissions here for the LDP mapping message from PE to P. Can you tell right away what could be the problem? 

Pay close attention to the packet size, 4814 bytes. That's awfully large for normal standards using a 1500 IP MTU + MPLS + VLAN tag + Ethernet. Some background information: We use an L2 MTU in the core always of 9192. This is basically the standard everywhere. However, there was a problem in this specific case, what could be the cause? 

When the wireless PTP installed, the technician used a L2 switch that could provide Passive PoE to the radio. When doing this, the technician did not raise the default L2 MTU from a value higher than the default of 1522. The routers on the ends of this link had an L2 MTU set of 9192, so they obviously were under the impression they could send much larger than 1522B frames. Because this was the routers' belief that they had a large L2 MTU to work with, they also believed they could send large IP MTU packets. That explains the 4814B packet being sent with many LDP label mappings.

The packet trace was basically my dead giveaway of what was happening. I realized what had been done by the techs in this situation with the L2 switch for PoE, I was able to make the adjustments to L2 MTU on said switch, and there was no longer an issue with the intermediate switch silently discarding large/jumbo frames. For someone new, you may wonder how this wasn't caught sooner - well this was strictly a backup wireless link as mentioned earlier. Let this serve as a reminder to always find a way to thoroughly test and monitor your redundant links, because when you need them you'd really like for them to work and perform well.

### Last Comments 
- To someone who hasn't ran into this before, you may have assumed there wouldn't be an issue because an OSPF adjacency comes up and that relies on MTU right? Well, yeah kind of. OSPF did come up, because in the DBD OSFP packet the MTU's of each router did match (9192B) But that is just a described value in the OSPF packets, it doesn't describe *actual forwarding ability*
- This can obviously happen with more than just LDP, take BGP (TCP/179) for example where this exact issue could also occur
- Remember L2-only devices will just silently discard frames too large for their L2 MTU. Only L3 devices will send you ICMP messages describing packet too large. 

### You made it this far 
- I cut off certain parts of the packet trace for data sanitization, I didn't have a proper lab set up for this example

*Thanks for reading*
