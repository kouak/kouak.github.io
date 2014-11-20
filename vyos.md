# What I'm trying to do with VyOS

### Network configuration
##### WAN Links :
* WAN1 (eth1 vif 10)
 * ipv4 : 1.2.3.4/22
 * gateway : 1.2.0.1
 * assigned via dhcp (the lease is long enough to consider anything static)
 * asym cable link : 30M down, 2M up
 * no ipv6
* WAN2 (eth1 vif 11)
 * ipv4 : 5.6.7.8/22
 * gateway : 5.6.4.1
 * assigned via dhcp (same as WAN1)
 * asym cable link : 30M down, 1M up
 * no ipv6
* WAN3 (eth1 vif 12 pppoe 0)
 * ipv4 : 9.10.11.12/32
 * gateway : 12.12.12.12 (directly connected)
 * routed ipv6 subnet :  2001:db8:12::/48
 * asym DSL link : 12M down / 1M up
* WAN4 (tun0)
 * HeNet tunnel over WAN1
 * tunnel source : 1.2.3.4 (WAN1)
 * tunnel destination : 200.200.200.200
 * routed ipv6 subnet : 2001:db8:200::/48

##### LAN Links :
* LAN (eth0 vif 200)
 * ipv4 : 10.36.200.1/24
 * ipv6 : fd36:cafe:beef:f200::1/52 (ULA)
* DMZ (eth0 vif 210)
 * ipv4 : 10.36.210.1/24
 * ipv6 : 2001:db8:12:c000::1/52

### Routing requirements and failover
##### IPv4
* Traffic from LAN should flow through WAN1/WAN2/WAN3 (source NATed to the WAN IP)
* Traffic from DMZ should flow through WAN3 only (source NATed to the WAN IP)
* Traffic from LAN should be routed through WAN1, WAN2 in case of failure of WAN1 and ultimately WAN3 in case of a failure of both WAN1 and WAN2
* Traffic to specific destinations should be routed via WAN2, no matter the status of WAN1 or WAN2
* Traffic to specific destionations should be routed via WAN1, no matter the status of WAN1 or WAN2 (e.g traffic to 200.200.200.200, HeNet tunnel destination)

##### IPv6
* Traffic from LAN should flow through WAN4 (translated via NPTv6 rules, I got this part working) *Note : I could ask for some kind of failover here, but this is far from being supported in VyOS as far as my knowledge goes*
* Traffic to/from DMZ should flow through WAN3 (with no translation, DMZ hosts v6 addressing lies in WAN4 routed subnet)

### Traffic shaping
* I need some kind of traffic shaping on top of that, both upway **and** downway but I haven't looked into it as of now.

### My need for PBR (and v6 PBR)
* To address my DMZ routing needs, I'm using PBR and v6 PBR (with a patch found on VyOS forum) as follow :
```
policy {
    ipv6-route DMZ6 {
        description "DMZ v6 policy routing"
        rule 10 {
            destination {
                address ::/0
            }
            set {
                table 50
            }
            source {
                address 2001:db8:12:c000::/52
            }
        }
    }
    route CAFAI-DMZ {
        description "DMZ routing policy"
        rule 10 {
            destination {
                address 0.0.0.0/0
            }
            set {
                table 50
            }
            source {
                address 10.36.20.0/24
            }
        }
    }
}
protocols {
    static {
        table 50 {
            interface-route6 ::/0 {
                next-hop-interface pppoe0 {
                }
            }
            route 0.0.0.0/0 {
                next-hop 12.12.12.12 {
                }
            }
        }
    }
}
```

### My Load Balacing conf
```
load-balancing {
    wan {
        disable-source-nat # NAT Rules are set in nat { }
        enable-local-traffic # I'd rather have local traffic failover'ed, but I'm not sure about the implications
        flush-connections
        interface-health eth1.10 { # WAN1
            failure-count 6
            nexthop dhcp
            success-count 3
            test 10 {
                resp-time 5
                target 8.8.4.4
                ttl-limit 1
                type ping
            }
        }
        interface-health eth1.11 { # WAN2
            failure-count 6
            nexthop dhcp
            success-count 3
            test 10 {
                resp-time 5
                target 8.8.8.8
                ttl-limit 1
                type ping
            }
        }
        interface-health pppoe0 { # WAN3
            failure-count 6
            nexthop 12.12.12.12
            success-count 3
            test 10 {
                resp-time 5
                target 12.12.12.12
                ttl-limit 1
            }
        }
        # Prevent test targets from being load balanced/failover'ed
        rule 10 {
            destination {
                address 12.12.12.12
            }
            exclude
            inbound-interface any
            protocol all
        }
        rule 11 {
            destination {
                address 8.8.8.8
            }
            exclude
            inbound-interface any
            protocol all
        }
        rule 12 {
            destination {
                address 8.8.4.4
            }
            exclude
            inbound-interface any
            protocol all
        }
        # Exclude specific destinations
        rule 20 {
            destination {
                address 50.0.0.1
            }
            exclude
            inbound-interface any
            protocol all
        }
        # Exclude HeNet tunnel endpoint
        rule 21 {
            destination {
                address 200.200.200.200
            }
            exclude
            inbound-interface any # I need to specifically exclude local traffic on this one to set up the HeNet tunnel properly
            protocol all
        }
        # Failover rule
        rule 100 {
            failover
            inbound-interface eth0.200
            interface eth1.10 {
                weight 10 # Preferred gateway
            }
            interface eth1.11 {
                weight 5 # First failover
            }
            interface pppoe0 {
                weight 1 # Second failover
            }
            protocol all
        }
    }
}
```

I have set up a bunch of static routes for excluded traffic :

```
protocols {
    static {
        interface-route6 ::/0 { # Default v6 route via HeNet tunnel
            next-hop-interface tun0 {
            }
        }
        # Test targets
        route 8.8.4.4/32 {
            next-hop 1.2.0.1 { # Route through WAN1
            }
        }
        route 8.8.8.8/32 {
            next-hop 5.6.4.1 { # Route through WAN2
            }
        }
        # Note : WAN3 test target is directly connected, I guess there is no need for a static rule here
        # Other excluded traffic
        route 200.200.200.200/32 {
            next-hop 1.2.0.1 { # HeNet goes through WAN1
            }
        }
        route 50.0.0.1/32 {
            next-hop 5.6.4.1 { # This one goes through WAN2
            }
        }
    }
}
```


        


