# Using BGP to resolve outbound connectivity with high availability in AWS VPC's

The story is that using both HTTP Proxies and NAT in AWS VPC are generally sub-optimal. Neither provide *quick* failover and both carry a high systems overhead for what *should* be a network engineers problem.

I put on my network engineering hat and decided to test using two VyOS hosts running BGP in the case of a typical multi-homed datacentre connection would be. The only significant difference in this example design versus a real production implementation is the use of IPSec tunnels rather than Amazon Direct Connects.

Once again thanks to Vultr for their fantastic public cloud which enables simple tests like this one with ISO loading for VyOS with minimal pain.

## Cloudformation

To run cloudformation via Ansible ensure;
1. You have your API creds in ~/.aws/credentials
1. Your running a recent Ansible
1. You change the variables in roles/cf_vpc/tasks/main.yml

Then kick-off Ansible using;

    $ ansible-playbook -i inventory/cloudformation site.yml 

## Configurations

At a high level what we are creating will look like this;

![Network Diagram](/aws-bgp-outbound.svg)

### Router1
```
 interfaces {
     ethernet eth0 {
         address dhcp
         address dhcpv6
     }
     loopback lo {
     }
     vti vti0 {
         address xxx.xxx.247.2/30
         description "VPC tunnel 1"
         mtu 1436
     }
     vti vti1 {
         address xxx.xxx.247.6/30
         description "VPC tunnel 2"
         mtu 1436
     }
 }
 nat {
     source {
         rule 1 {
             outbound-interface eth0
             protocol all
             source {
                 address xxx.xxx.0.0/24
             }
             translation {
                 address masquerade
             }
         }
     }
 }
 protocols {
     bgp XXXXXX {
         neighbor xxx.xxx.247.1 {
             remote-as XXXXXX
             soft-reconfiguration {
                 inbound
             }
             timers {
                 holdtime 30
                 keepalive 30
             }
         }
         neighbor xxx.xxx.247.5 {
             remote-as XXXXXX
             soft-reconfiguration {
                 inbound
             }
             timers {
                 holdtime 30
                 keepalive 30
             }
         }
         network xxx.xxx.0.0/0 {
         }
         redistribute {
             static {
                 metric 10
             }
         }
     }
 }
 service {
     ssh {
         listen-address xxx.xxx.0.0
     }
 }
 system {
     config-management {
         commit-revisions 20
     }
     console {
         device ttyS0 {
             speed 9600
         }
     }
     host-name xxxxxx
     login xxxxxx
         user xxxxxx {
             authentication {
                 encrypted-password xxxxxx
             }
             level admin
         }
     }
     ntp {
         server xxxxx.tld {
         }
         server xxxxx.tld {
         }
         server xxxxx.tld {
         }
     }
     package {
         auto-sync 1
         repository community {
             components main
             distribution helium
             password xxxxxx
             url http://packages.vyos.net/vyos
             username xxxxxx
         }
     }
     syslog {
         global {
             facility all {
                 level notice
             }
             facility protocols {
                 level debug
             }
         }
     }
     time-zone UTC
 }
 vpn {
     ipsec {
         esp-group AWS {
             compression disable
             lifetime 3600
             mode tunnel
             pfs enable
             proposal 1 {
                 encryption aes128
                 hash sha1
             }
         }
         ike-group AWS {
             dead-peer-detection {
                 action restart
                 interval 15
                 timeout 30
             }
             lifetime 28800
             proposal 1 {
                 dh-group 2
                 encryption aes128
                 hash sha1
             }
         }
         ipsec-interfaces {
             interface eth0
         }
         site-to-site {
             peer xxxxx.tld {
                 authentication {
                     mode pre-shared-secret
                     pre-shared-secret xxxx
                 }
                 description "VPC tunnel 1"
                 ike-group AWS
                 local-address xxx.xxx.251.149
                 vti {
                     bind vti0
                     esp-group AWS
                 }
             }
             peer xxxxx.tld {
                 authentication {
                     mode pre-shared-secret
                     pre-shared-secret xxxx
                 }
                 description "VPC tunnel 2"
                 ike-group AWS
                 local-address xxx.xxx.251.149
                 vti {
                     bind vti1
                     esp-group AWS
                 }
             }
         }
     }
 }
```
### Router2

Here router2 acts a backup ISP by setting its route advertisements to use weight 65535. This causes router1's advertisements to be used *always* unless they are withdrawn for maintenance (planned or unplanned). This pattern can be repeated how ever many times you wish to add N+1, N+2 ISP's to your VPC.

**Alternatively** you can also remove the policy-map from router2 and instead both routers will be equal. This will cause traffic to follow a round-robin exit path and effectively act as a crude "load balanced" connection with the added redundancy of tolerating a single router withdrawing its routes.

```
 interfaces {
     ethernet eth0 {
         address dhcp
         address dhcpv6
     }
     loopback lo {
     }
     vti vti0 {
         address xxx.xxx.247.26/30
         description "VPC tunnel 1"
         mtu 1436
     }
     vti vti1 {
         address xxx.xxx.247.30/30
         description "VPC tunnel 2"
         mtu 1436
     }
 }
 nat {
     source {
         rule 1 {
             outbound-interface eth0
             protocol all
             source {
                 address xxx.xxx.0.0/24
             }
             translation {
                 address masquerade
             }
         }
     }
 }
 policy {
     route-map def-route {
         rule 1 {
             action permit
             match {
                 metric 20
             }
             set {
                 weight 65536
             }
         }
     }
 }
 protocols {
     bgp XXXXXX {
         neighbor xxx.xxx.247.25 {
             remote-as XXXXXX
             soft-reconfiguration {
                 inbound
             }
             timers {
                 holdtime 30
                 keepalive 30
             }
         }
         neighbor xxx.xxx.247.29 {
             remote-as XXXXXX
             soft-reconfiguration {
                 inbound
             }
             timers {
                 holdtime 30
                 keepalive 30
             }
         }
         network xxx.xxx.0.0/0 {
             route-map def-route
         }
         redistribute {
             static {
                 metric 20
             }
         }
     }
 }
 service {
     ssh {
         listen-address xxx.xxx.0.0
     }
 }
 system {
     config-management {
         commit-revisions 20
     }
     console {
         device ttyS0 {
             speed 9600
         }
     }
     host-name xxxxxx
     login xxxxxx
         user xxxxxx {
             authentication {
                 encrypted-password xxxxxx
             }
             level admin
         }
     }
     ntp {
         server xxxxx.tld {
         }
         server xxxxx.tld {
         }
         server xxxxx.tld {
         }
     }
     package {
         auto-sync 1
         repository community {
             components main
             distribution helium
             password xxxxxx
             url http://packages.vyos.net/vyos
             username xxxxxx
         }
     }
     syslog {
         global {
             facility all {
                 level notice
             }
             facility protocols {
                 level debug
             }
         }
     }
     time-zone UTC
 }
 vpn {
     ipsec {
         esp-group AWS {
             compression disable
             lifetime 3600
             mode tunnel
             pfs enable
             proposal 1 {
                 encryption aes128
                 hash sha1
             }
         }
         ike-group AWS {
             dead-peer-detection {
                 action restart
                 interval 15
                 timeout 30
             }
             lifetime 28800
             proposal 1 {
                 dh-group 2
                 encryption aes128
                 hash sha1
             }
         }
         ipsec-interfaces {
             interface eth0
         }
         site-to-site {
             peer xxxxx.tld {
                 authentication {
                     mode pre-shared-secret
                     pre-shared-secret xxxx
                 }
                 description "VPC tunnel 1"
                 ike-group AWS
                 local-address xxx.xxx.232.83
                 vti {
                     bind vti0
                     esp-group AWS
                 }
             }
             peer xxxxx.tld {
                 authentication {
                     mode pre-shared-secret
                     pre-shared-secret xxxx
                 }
                 description "VPC tunnel 2"
                 ike-group AWS
                 local-address xxx.xxx.232.83
                 vti {
                     bind vti1
                     esp-group AWS
                 }
             }
         }
     }
 }
```
