---
layout: post
title: Poor man's BFD - JunOS
--- 

Title isn't 100% accurate, because I'm not making this post of code snippets because I'm poor and can't run REAL BFD. Rather, recently added a transit provider who refused to config BFD on our BGP sessions. LACP in a LAG/port-channel also wasn't an option, too many L2 devices between my router and theirs in the transport to make it useful. 

---

So here we are, configuring RPM and event-options. I'm just going to drop the code snippets and explain it. 

```
[edit]
root@R1# show services 
rpm {
    probe bgp-mon {
        test icmp-test {
            probe-type icmp-ping;
            target address 192.168.0.0;
            probe-count 5;
            probe-interval 5;
            ttl 1;
            thresholds {
                total-loss 5;
            }
        }
    }
}

root@R1# show event-options 
policy clear-bgp-neighbor-ping-failure {
    events [ ping_test_failed ping_probe_failed ];
    attributes-match {
        ping_test_failed.test-owner matches bgp-mon;
        ping_test_failed.test-name matches icmp-test;
    }
    then {
        execute-commands {
            commands {
                "clear bgp neighbor 192.168.0.0";
            }
        }
    }
}
```

## RPM 
In JunOS, RPM (Real-Time Performance Monitoring) enables you to configure active probes to track and monitor traffic across the network and to investigate network problems.

We are using a basic RPM config with ICMP echo-request probes, and a successful echo-reply means the probe was successful. If the ping times out, the host target (192.168.0.0) on the other end fails the probe test. 

* probe-count 5;           # number of probes we sent in our ping test 
* probe-interval 5;        # number of seconds to wait between each test 
* ttl 1;                   # important, I want the test to only succeed if the router is 1 hop away
* thresholds total-loss 5; # if all 5 pings in the test are failed, fail this rpm test

## Event Options 
Events in JunOS are received by a process called eventd, and the events can come from other processes such as rpd (routing protocol process) or mgd (management process). Event policies can be configured to use events as triggers, and to perform some sort of action after the trigger. Think of the JunOS eventd service as being fed by other management, routing, and infrastructure processes. Eventd relies on syslog entries or SNMP traps. 

Our event policy watches for a ping_test_failed event with RPM owner bgp-mon and test name icmp-test, sounds familiar. If those events are met, we clear the bgp session with neighbor 192.168.0.0. 

---

## Verification 
We configured the RPM test and the event policy. Let's see how it works. To simulate an outage, I deactivated the far-end interface of the target (192.168.0.0). Normally, without a link-down event BGP would take up to a full 90s hold-time to tear down the session. With RPM configured, we notice within a 5s interval with our config. Below, with some verbose logging you can see the 5 ping probes fail, so the test fails and our event policy tears down our bgp session as configured. 

```
set system syslog file daemon-info.log daemon info" 
```

```
root@R1# run show log daemon-info.log 
Feb 12 17:30:14  R1 rmopd[9650]: PING_PROBE_FAILED: pingCtlOwnerIndex = bgp-mon, pingCtlTestName = icmp-test
Feb 12 17:30:19  R1 rmopd[9650]: PING_PROBE_FAILED: pingCtlOwnerIndex = bgp-mon, pingCtlTestName = icmp-test
Feb 12 17:30:19  R1 rmopd[9650]: PING_TEST_FAILED: pingCtlOwnerIndex = bgp-mon, pingCtlTestName = icmp-test
Feb 12 17:30:25  R1 rmopd[9650]: PING_PROBE_FAILED: pingCtlOwnerIndex = bgp-mon, pingCtlTestName = icmp-test
Feb 12 17:30:30  R1 rmopd[9650]: PING_PROBE_FAILED: pingCtlOwnerIndex = bgp-mon, pingCtlTestName = icmp-test
Feb 12 17:30:30  R1 rpd[7597]: bgp_peer_mgmt_clear:8430: NOTIFICATION sent to 192.168.0.0 (External AS 64511): code 6 (Cease) subcode 4 (Administratively Reset), Reason: Management session cleared BGP neighbor
Feb 12 17:30:40  R1 rmopd[9650]: PING_TEST_COMPLETED: pingCtlOwnerIndex = bgp-mon, pingCtlTestName = icmp-test
```

If you want to watch it in real-time, use "monitor start daemon-info.log"

---

### Man, sure would be nice if there was a protocol that did all this without special on-box work! 
That protocol is BFD (Bidirectional Forwarding Detection) and it must be a HUGE operational toll for some transit providers to configure it. It's only been around <a href="https://tools.ietf.org/html/rfc5880" target="_blank">for a decade.</a>

### References
<a href="https://www.juniper.net/documentation/en_US/junos/topics/concept/junos-script-automation-event-notifications-and-policy-overview.html
" target="_blank">https://www.juniper.net/documentation/en_US/junos/topics/concept/junos-script-automation-event-notifications-and-policy-overview.html</a>

<a href="https://kb.juniper.net/InfoCenter/index?page=content&id=KB35533&actp=RSS" target="_blank">https://kb.juniper.net/InfoCenter/index?page=content&id=KB35533&actp=RSS</a>

<a href="https://www.juniper.net/documentation/en_US/junos/topics/topic-map/real-time-performance-monitoring.html#id-understanding-real-time-performance-monitoring-on-switches
" target="_blank">https://www.juniper.net/documentation/en_US/junos/topics/topic-map/real-time-performance-monitoring.html#id-understanding-real-time-performance-monitoring-on-switches</a>

<a href="https://www.juniper.net/documentation/en_US/junos/topics/topic-map/real-time-performance-monitoring.html#jd0e4418" target="_blank">https://www.juniper.net/documentation/en_US/junos/topics/topic-map/real-time-performance-monitoring.html#jd0e4418</a>