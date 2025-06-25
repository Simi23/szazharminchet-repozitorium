---
title: E$-2S
description: 1st solve
published: true
date: 2025-06-25T07:42:02.792Z
tags: cisco
editor: markdown
dateCreated: 2025-06-25T07:36:06.033Z
---

# E$-2S 1st solve

<details>
<summary>CLOUDFW</summary>
  
  ```
: Saved

:
: Serial Number: 9AJLXAFPGV3
: Hardware:   ASAv, 2048 MB RAM, CPU Pentium II 2900 MHz
:
ASA Version 9.20(2)2
!
hostname CLOUDFW
domain-name isp.denmark
enable password ***** pbkdf2
service-module 0 keepalive-timeout 4
service-module 0 keepalive-counter 6
names
no mac-address auto

!
interface GigabitEthernet0/0
  no nameif
  no security-level
  no ip address
!
interface GigabitEthernet0/0.10
  vlan 10
  nameif inside
  security-level 100
  ip address 192.168.255.2 255.255.255.252
!
interface GigabitEthernet0/0.100
  vlan 100
  nameif outside
  security-level 0
  ip address 8.8.8.10 255.255.255.0
!
interface GigabitEthernet0/1
  nameif dmz
  security-level 50
  ip address 192.168.50.254 255.255.255.0
!
interface Management0/0
  no management-only
  shutdown
  no nameif
  no security-level
  no ip address
!
ftp mode passive
dns server-group DefaultDNS
  domain-name isp.denmark
no object-group-search access-control
object network tunnel
  subnet 10.0.0.0 255.255.255.0
object network hq
  subnet 172.16.0.0 255.255.0.0
object network cpe4
  subnet 192.168.14.0 255.255.254.0
object network cpe5
  subnet 172.32.20.0 255.255.252.0
object network inside
  subnet 0.0.0.0 0.0.0.0
object network dmz
  subnet 192.168.50.0 255.255.255.0
object-group network LOCAL
  network-object object tunnel
  network-object object hq
  network-object object cpe4
  network-object object cpe5
object-group network internal
  network-object object hq
  network-object object cpe4
  network-object object cpe5
access-group cloudsrv out interface dmz
access-list cloudsrv extended permit ip object-group internal object dmz
pager lines 23
mtu inside 1500
mtu outside 1500
mtu dmz 1500
no failover
no failover wait-disable
no monitor-interface inside
no monitor-interface outside
icmp unreachable rate-limit 1 burst-size 1
no asdm history enable
arp timeout 14400
no arp permit-nonconnected
arp rate-limit 8192
!
object network inside
  nat (inside,outside) dynamic interface
object network dmz
  nat (dmz,outside) dynamic interface
router ospf 10
  network 192.168.50.0 255.255.255.0 area 0
  network 192.168.255.0 255.255.255.252 area 0
  log-adj-changes
  default-information originate
!
router bgp 65003
  bgp log-neighbor-changes
!
route outside 0.0.0.0 0.0.0.0 8.8.8.8 1
timeout xlate 3:00:00
timeout pat-xlate 0:00:30
timeout conn 1:00:00 half-closed 0:10:00 udp 0:02:00 sctp 0:02:00 icmp 0:00:02
timeout sunrpc 0:10:00 h323 0:05:00 h225 1:00:00 mgcp 0:05:00 mgcp-pat 0:05:00
timeout sip 0:30:00 sip_media 0:02:00 sip-invite 0:03:00 sip-disconnect 0:02:00
timeout sip-provisional-media 0:02:00 uauth 0:05:00 absolute
timeout tcp-proxy-reassembly 0:01:00
timeout floating-conn 0:00:00
timeout conn-holddown 0:00:15
timeout igp stale-route 0:01:10
user-identity default-domain LOCAL
aaa authentication ssh console LOCAL
aaa authorization exec LOCAL auto-enable
aaa authentication login-history
no snmp-server location
no snmp-server contact
crypto ipsec security-association pmtu-aging infinite
crypto ca trustpoint _SmartCallHome_ServerCA
  no validation-usage
  crl configure
crypto ca trustpoint _SmartCallHome_ServerCA2
  no validation-usage
  crl configure
crypto ca trustpool policy
  auto-import
crypto ca certificate chain _SmartCallHome_ServerCA
  certificate ca 0a0142800000014523c844b500000002
    30820560 30820348 a0030201 0202100a 01428000 00014523 c844b500 00000230
    0d06092a 864886f7 0d01010b 0500304a 310b3009 06035504 06130255 53311230
    10060355 040a1309 4964656e 54727573 74312730 25060355 0403131e 4964656e
    54727573 7420436f 6d6d6572 6369616c 20526f6f 74204341 2031301e 170d3134
    30313136 31383132 32335a17 0d333430 31313631 38313232 335a304a 310b3009
    06035504 06130255 53311230 10060355 040a1309 4964656e 54727573 74312730
    25060355 0403131e 4964656e 54727573 7420436f 6d6d6572 6369616c 20526f6f
    74204341 20313082 0222300d 06092a86 4886f70d 01010105 00038202 0f003082
    020a0282 020100a7 5019de3f 993dd433 46f16f51 6182b2a9 4f8f6789 5d84d953
    dd0c28d9 d7f0ffae 95437299 f9b55d7c 8ac142e1 315074d1 810d7ccd 9b21ab43
    e2acad5e 866ef309 8a1f5a32 bda2eb94 f9e85c0a ecff98d2 af71b3b4 539f4e87
    ef92bcbd ec4f3230 884b175e 57c453c2 f602978d d9622bbf 241f628d dfc3b829
    4b49783c 93608822 fc99da36 c8c2a2d4 2c540067 356e73bf 0258f0a4 dde5b0a2
    267acae0 36a51916 f5fdb7ef ae3f40f5 6d5a04fd ce34ca24 dc74231b 5d331312
    5dc40125 f630dd02 5d9fe0d5 47bdb4eb 1ba1bb49 49d89f5b 02f38ae4 2490e462
    4f4fc1af 8b0e7417 a8d17288 6a7a0149 ccb44679 c617b1da 981e0759 fa752185
    65dd9056 cefbaba5 609dc49d f952b08b bd87f98f 2b230a23 763bf733 e1c900f3
    69f94ba2 e04ebc7e 93398407 f744707e fe075ae5 b1acd118 ccf235e5 494908ca
    56c93dfb 0f187d8b 3bc113c2 4d8fc94f 0e37e91f a10e6adf 622ecb35 0651792c
    c82538f4 fa4ba789 5c9cd2e3 0d39864a 747cd559 87c23f4e 0c5c52f4 3df75282
    f1eaa3ac fd49341a 28f34188 3a13eee8 deff991d 5fbacbe8 1ef2b950 60c031d3
    73e5efbe a0ed330b 74be2020 c4676cf0 08037a55 807f464e 96a7f41e 3ee1f6d8
    09e13364 2b63d732 5e9ff9c0 7b0f786f 97bc939a f99c1290 787a8087 15d77274
    9c557478 b1bae16e 7004ba4f a0ba68c3 7bff31f0 733d3d94 2ab10b41 0ea0fe4d
    88656b79 33b4d702 03010001 a3423040 300e0603 551d0f01 01ff0404 03020106
    300f0603 551d1301 01ff0405 30030101 ff301d06 03551d0e 04160414 ed4419c0
    d3f0068b eea47bbe 42e72654 c88e3676 300d0609 2a864886 f70d0101 0b050003
    82020100 0dae9032 f6a64b7c 44761961 1e2728cd 5e54ef25 bce30890 f929d7ae
    6808e194 0058ef2e 2e7e5352 8cb65c07 ea88ba99 8b5094d7 8280df61 090093ad
    0d14e6ce c1f23794 78b05f9c b3a273b8 8f059338 cd8d3eb0 b8fbc0cf b1f2ec2d
    2d1bccec aa9ab3aa 60821b2d 3bc3843d 578a961e 9c75b8d3 30cd6008 8390d38e
    54f14d66 c05d7403 40a3ee85 7ec21f77 9c06e8c1 a7185d52 95edc9dd 259e6dfa
    a9eda33a 34d0597b daed50f3 35bfedeb 144d31c7 60f4daf1 879ce248 e2c6c537
    fb0610fa 75596631 4729da76 9a1ce982 aeef9ab9 51f78823 9a699562 3ce55580
    36d75402 fff1b95d ced4236f d845844a 5b65ef89 0cdd14a7 20cb18a5 25b40df9
    01f0a2d2 f400c874 8ea12a48 8e65db13 c4e22517 7debbe87 5b172054 51934a53
    030bec5d ca33ed62 fd45c72f 5bdc58a0 8039e6fa d7fe1314 a6ed3d94 4a4274d4
    c3775973 cd8f46be 5538effa e89132ea 97580422 de38c3cc bc6dc933 3a6a0a69
    3fa0c8ea 728f8c63 8623bd6d 3c969e95 e0494caa a2b92a1b 9c368178 edc3e846
    e2265944 751ed975 8951cd10 849d6160 cb5df997 224d8e98 e6e37ff6 5bbbaecd
    ca4a816b 5e0bf351 e1742be9 7e27a7d9 99494ef8 a580db25 0f1c6362 8ac93367
    6b3c1083 c6addea8 cd168e8d f0073771 9ff2abfc 41f5c18b ec00375d 09e54e80
    effab15c 3806a51b 4ae1dc38 2d3cdcab 1f901ad5 4a9ceed1 706cccee f457f818
    ba846e87
  quit
crypto ca certificate chain _SmartCallHome_ServerCA2
  certificate ca 0509
    308205b7 3082039f a0030201 02020205 09300d06 092a8648 86f70d01 01050500
    3045310b 30090603 55040613 02424d31 19301706 0355040a 13105175 6f566164
    6973204c 696d6974 6564311b 30190603 55040313 1251756f 56616469 7320526f
    6f742043 41203230 1e170d30 36313132 34313832 3730305a 170d3331 31313234
    31383233 33335a30 45310b30 09060355 04061302 424d3119 30170603 55040a13
    1051756f 56616469 73204c69 6d697465 64311b30 19060355 04031312 51756f56
    61646973 20526f6f 74204341 20323082 0222300d 06092a86 4886f70d 01010105
    00038202 0f003082 020a0282 0201009a 18ca4b94 0d002daf 03298af0 0f81c8ae
    4c19851d 089fab29 4485f32f 81ad321e 9046bfa3 86261a1e fe7e1c18 3a5c9c60
    172a3a74 8333307d 615411cb edabe0e6 d2a27ef5 6b6f18b7 0a0b2dfd e93eef0a
    c6b310e9 dcc24617 f85dfda4 daff9e49 5a9ce633 e62496f7 3fba5b2b 1c7a35c2
    d667feab 66508b6d 28602bef d760c3c7 93bc8d36 91f37ff8 db1113c4 9c7776c1
    aeb7026a 817aa945 83e205e6 b956c194 378f4871 6322ec17 6507958a 4bdf8fc6
    5a0ae5b0 e35f5e6b 11ab0cf9 85eb44e9 f80473f2 e9fe5c98 8cf573af 6bb47ecd
    d45c022b 4c39e1b2 95952d42 87d7d5b3 9043b76c 13f1dedd f6c4f889 3fd175f5
    92c391d5 8a88d090 ecdc6dde 89c26571 968b0d03 fd9cbf5b 16ac92db eafe797c
    adebaff7 16cbdbcd 252be51f fb9a9fe2 51cc3a53 0c48e60e bdc9b476 0652e611
    13857263 0304e004 362b2019 02e874a7 1fb6c956 66f07525 dc67c10e 616088b3
    3ed1a8fc a3da1db0 d1b12354 df44766d ed41d8c1 b222b653 1cdf351d dca1772a
    31e42df5 e5e5dbc8 e0ffe580 d70b63a0 ff33a10f ba2c1515 ea97b3d2 a2b5bef2
    8c961e1a 8f1d6ca4 6137b986 7333d797 969e237d 82a44c81 e2a1d1ba 675f9507
    a32711ee 16107bbc 454a4cb2 04d2abef d5fd0c51 ce506a08 31f991da 0c8f645c
    03c33a8b 203f6e8d 673d3ad6 fe7d5b88 c95efbcc 61dc8b33 77d34432 35096204
    921610d8 9e2747fb 3b21e3f8 eb1d5b02 03010001 a381b030 81ad300f 0603551d
    130101ff 04053003 0101ff30 0b060355 1d0f0404 03020106 301d0603 551d0e04
    1604141a 8462bc48 4c332504 d4eed0f6 03c41946 d1946b30 6e060355 1d230467
    30658014 1a8462bc 484c3325 04d4eed0 f603c419 46d1946b a149a447 3045310b
    30090603 55040613 02424d31 19301706 0355040a 13105175 6f566164 6973204c
    696d6974 6564311b 30190603 55040313 1251756f 56616469 7320526f 6f742043
    41203282 02050930 0d06092a 864886f7 0d010105 05000382 0201003e 0a164d9f
    065ba8ae 715d2f05 2f67e613 4583c436 f6f3c026 0c0db547 645df8b4 72c946a5
    03182755 89787d76 ea963480 1720dce7 83f88dfc 07b8da5f 4d2e67b2 84fdd944
    fc775081 e67cb4c9 0d0b7253 f8760707 4147960c fbe08226 93558cfe 221f6065
    7c5fe726 b3f73290 9850d437 7155f692 2178f795 79faf82d 26876656 3077a637
    78335210 58ae3f61 8ef26ab1 ef187e4a 5963ca8d a256d5a7 2fbc561f cf39c1e2
    fb0aa815 2c7d4d7a 63c66c97 443cd26f c34a170a f890d257 a21951a5 2d9741da
    074fa950 da908d94 46e13ef0 94fd1000 38f53be8 40e1b46e 561a20cc 6f588ded
    2e458fd6 e9933fe7 b12cdf3a d6228cdc 84bb226f d0f8e4c6 39e90488 3cc3baeb
    557a6d80 9924f56c 01fbf897 b0945beb fdd26ff1 77680d35 6423acb8 55a103d1
    4d4219dc f8755956 a3f9a849 79f8af0e b911a07c b76aed34 d0b62662 381a870c
    f8e8fd2e d3907f07 912a1dd6 7e5c8583 99b03808 3fe95ef9 3507e4c9 626e577f
    a75095f7 bac89be6 8ea201c5 d666bf79 61f33c1c e1b9825c 5da0c3e9 d848bd19
    a2111419 6eb2861b 683e4837 1a88b75d 965e9cc7 ef276208 e291195c d2f121dd
    ba174282 97718153 31a99ff6 7d62bf72 e1a3931d cc8a265a 0938d0ce d70d8016
    b478a53a 874c8d8a a5d54697 f22c10b9 bc5422c0 01506943 9ef4b2ef 6df8ecda
    f1e3b1ef df918f54 2a0b25c1 2619c452 100565d5 8210eac2 31cd2e
  quit
telnet timeout 10
ssh stack ciscossh
ssh stricthostkeycheck
ssh timeout 5
ssh key-exchange group dh-group14-sha256
ssh 0.0.0.0 0.0.0.0 inside
ssh 0.0.0.0 0.0.0.0 outside
ssh 0.0.0.0 0.0.0.0 dmz
console timeout 0
console serial
threat-detection basic-threat
threat-detection statistics access-list
no threat-detection statistics tcp-intercept
dynamic-access-policy-record DfltAccessPolicy
username Administrator password ***** pbkdf2 privilege 15
!
class-map inspection_default
  match default-inspection-traffic
!
!
policy-map type inspect dns preset_dns_map
  parameters
  message-length maximum client auto
  message-length maximum 512
  no tcp-inspection
policy-map global_policy
  class inspection_default
  inspect ip-options
  inspect netbios
  inspect rtsp
  inspect sunrpc
  inspect tftp
  inspect dns preset_dns_map
  inspect ftp
  inspect h323 h225
  inspect h323 ras
  inspect rsh
  inspect esmtp
  inspect sqlnet
  inspect sip
  inspect skinny
  inspect icmp
policy-map type inspect dns migrated_dns_map_2
  parameters
  message-length maximum client auto
  message-length maximum 512
  no tcp-inspection
policy-map type inspect dns migrated_dns_map_1
  parameters
  message-length maximum client auto
  message-length maximum 512
  no tcp-inspection
!
service-policy global_policy global
prompt hostname context
no call-home reporting anonymous
call-home
  profile License
  destination address http https://tools.cisco.com/its/service/oddce/services/DDCEService
  destination transport-method http
  profile CiscoTAC-1
  no active
  destination address http https://tools.cisco.com/its/service/oddce/services/DDCEService
  destination address email callhome@cisco.com
  destination transport-method http
  subscribe-to-alert-group diagnostic
  subscribe-to-alert-group environment
  subscribe-to-alert-group inventory periodic monthly
  subscribe-to-alert-group configuration periodic monthly
  subscribe-to-alert-group telemetry periodic daily
Cryptochecksum:93b02cfe49ea75f545e5b4d61e545b76
: end
	
  ```
</details>

<details>
<summary>CLOUDSW</summary>
  
  ```
  Building configuration...

Current configuration : 3091 bytes
!
! Last configuration change at 14:45:55 UTC Tue Jun 17 2025
!
version 15.2
service timestamps debug datetime msec
service timestamps log datetime msec
no service password-encryption
service compress-config
!
hostname CLOUDSW
!
boot-start-marker
boot-end-marker
!
!
no logging console
!
username Administrator privilege 15 secret 5 $1$lGJl$pY5MXHGto16URdiLWOQMJ.
no aaa new-model
!
!
!
!
!
!
!
!
no ip domain-lookup
ip domain-name isp.denmark
ip cef
no ipv6 cef
!
!
!
spanning-tree mode pvst
spanning-tree extend system-id
!
!
!
!
!
!
!
!
!
!
!
!
!
!
!
interface GigabitEthernet0/0
switchport trunk encapsulation dot1q
switchport mode trunk
negotiation auto
!
interface GigabitEthernet0/1
switchport access vlan 100
switchport mode access
negotiation auto
!
interface GigabitEthernet0/2
switchport trunk encapsulation dot1q
switchport mode trunk
negotiation auto
!
interface GigabitEthernet0/3
negotiation auto
!
ip forward-protocol nd
!
ip http server
ip http secure-server
!
ip ssh version 2
ip ssh server algorithm encryption aes128-ctr aes192-ctr aes256-ctr
ip ssh client algorithm encryption aes128-ctr aes192-ctr aes256-ctr
!
!
!
!
!
!
control-plane
!
banner exec ^C
IOSv - Cisco Systems Confidential -

Supplemental End User License Restrictions

This IOSv software is provided AS-IS without warranty of any kind. Under no circumstances may this software be used separate from the Cisco Modeling Labs Software that this software was provided with, or deployed or used as part of a production environment.

By using the software, you agree to abide by the terms and conditions of the Cisco End User License Agreement at http://www.cisco.com/go/eula. Unauthorized use or distribution of this software is expressly prohibited.
^C
banner incoming ^C
IOSv - Cisco Systems Confidential -

Supplemental End User License Restrictions

This IOSv software is provided AS-IS without warranty of any kind. Under no circumstances may this software be used separate from the Cisco Modeling Labs Software that this software was provided with, or deployed or used as part of a production environment.

By using the software, you agree to abide by the terms and conditions of the Cisco End User License Agreement at http://www.cisco.com/go/eula. Unauthorized use or distribution of this software is expressly prohibited.
^C
banner login ^C
IOSv - Cisco Systems Confidential -

Supplemental End User License Restrictions

This IOSv software is provided AS-IS without warranty of any kind. Under no circumstances may this software be used separate from the Cisco Modeling Labs Software that this software was provided with, or deployed or used as part of a production environment.

By using the software, you agree to abide by the terms and conditions of the Cisco End User License Agreement at http://www.cisco.com/go/eula. Unauthorized use or distribution of this software is expressly prohibited.
^C
!
line con 0
exec-timeout 0 0
logging synchronous
line aux 0
line vty 0 4
exec-timeout 0 0
login local
transport input ssh
line vty 5 15
login local
transport input ssh
!
!
end
  
  ```
</details>

<details>
<summary>CORE-PE</summary>
    
  ```
  
  
  ```
</details>

<details>
<summary>PE-01</summary>
    
  ```
  
  
  ```
</details>

<details>
<summary>PE-02</summary>
    
  ```
  Building configuration...

Current configuration : 3919 bytes
!
! Last configuration change at 14:45:55 UTC Tue Jun 17 2025
!
version 15.9
service timestamps debug datetime msec
service timestamps log datetime msec
no service password-encryption
!
hostname PE-02
!
boot-start-marker
boot-end-marker
!
!
no logging console
!
no aaa new-model
!
!
!
mmi polling-interval 60
no mmi auto-configure
no mmi pvc
mmi snmp-timeout 180
!
!
!
!
!
!
!
!
!
!
!
no ip domain lookup
ip domain name isp.denmark
ip cef
no ipv6 cef
!
multilink bundle-name authenticated
!
!
!
!
username Administrator privilege 15 secret 9 $9$xa3g0B9XoQGX8P$trYkr2jrR0bN0w.TEl5/aQRVCiBXaPEN/Iwat1eG3to
!
redundancy
!
!
!
!
!
!
!
!
!
!
!
!
!
!
!
interface Loopback0
ip address 30.30.30.30 255.255.255.255
!
interface GigabitEthernet0/0
ip address 10.252.0.6 255.255.255.252
duplex auto
speed auto
media-type rj45
!
interface GigabitEthernet0/1
ip address 80.200.200.13 255.255.255.252
duplex auto
speed auto
media-type rj45
!
interface GigabitEthernet0/2
ip address 80.200.200.9 255.255.255.252
duplex auto
speed auto
media-type rj45
!
interface GigabitEthernet0/3
no ip address
shutdown
duplex auto
speed auto
media-type rj45
!
router bgp 8654
bgp log-neighbor-changes
network 30.30.30.30 mask 255.255.255.255
network 80.200.200.8 mask 255.255.255.252
network 80.200.200.12 mask 255.255.255.252
neighbor EBGP peer-group
neighbor EBGP password Passw0rd!
neighbor EBGP route-map DEFAULT out
neighbor 10.252.0.5 remote-as 8654
neighbor 80.200.200.10 remote-as 65001
neighbor 80.200.200.10 peer-group EBGP
neighbor 80.200.200.14 remote-as 65003
neighbor 80.200.200.14 peer-group EBGP
!
ip forward-protocol nd
!
!
no ip http server
no ip http secure-server
ip ssh version 2
!
!
ip prefix-list DEFAULT seq 5 permit 0.0.0.0/0
ipv6 ioam timestamp
!
route-map DEFAULT permit 10
match ip address prefix-list DEFAULT
!
!
!
control-plane
!
banner exec ^C
**************************************************************************
* IOSv is strictly limited to use for evaluation, demonstration and IOS  *
* education. IOSv is provided as-is and is not supported by Cisco's      *
* Technical Advisory Center. Any use or disclosure, in whole or in part, *
* of the IOSv Software or Documentation to any third party for any       *
* purposes is expressly prohibited except as otherwise authorized by     *
* Cisco in writing.                                                      *
**************************************************************************^C
banner incoming ^C
**************************************************************************
* IOSv is strictly limited to use for evaluation, demonstration and IOS  *
* education. IOSv is provided as-is and is not supported by Cisco's      *
* Technical Advisory Center. Any use or disclosure, in whole or in part, *
* of the IOSv Software or Documentation to any third party for any       *
* purposes is expressly prohibited except as otherwise authorized by     *
* Cisco in writing.                                                      *
**************************************************************************^C
banner login ^C
**************************************************************************
* IOSv is strictly limited to use for evaluation, demonstration and IOS  *
* education. IOSv is provided as-is and is not supported by Cisco's      *
* Technical Advisory Center. Any use or disclosure, in whole or in part, *
* of the IOSv Software or Documentation to any third party for any       *
* purposes is expressly prohibited except as otherwise authorized by     *
* Cisco in writing.                                                      *
**************************************************************************^C
!
line con 0
exec-timeout 0 0
logging synchronous
line aux 0
line vty 0 4
exec-timeout 0 0
login local
transport input ssh
line vty 5 15
login local
transport input ssh
!
no scheduler allocate
!
end
  
  ```
</details>

<details>
<summary>CPE-01</summary>
    
  ```
  Building configuration...

Current configuration : 4761 bytes
!
! Last configuration change at 14:45:56 UTC Tue Jun 17 2025
!
version 15.9
service timestamps debug datetime msec
service timestamps log datetime msec
no service password-encryption
!
hostname CPE-01
!
boot-start-marker
boot-end-marker
!
!
vrf definition herningco
rd 65001:100
!
address-family ipv4
exit-address-family
!
no logging console
!
no aaa new-model
!
!
!
mmi polling-interval 60
no mmi auto-configure
no mmi pvc
mmi snmp-timeout 180
!
!
!
!
!
!
!
!
!
!
!
ip domain name isp.denmark
ip cef
no ipv6 cef
!
multilink bundle-name authenticated
!
!
!
!
username Administrator privilege 15 secret 9 $9$IoS8zQDAk7GeFf$7F2K4vPQsHVRxomLICkQ6tI8Rk0qMdoIq2jPxRb2.nE
!
redundancy
!
!
!
!
crypto ikev2 proposal IKE-PROP
encryption aes-cbc-256
integrity sha256
group 14
!
crypto ikev2 policy IKE-POL
match fvrf any
match address local 1.1.1.1
match address local 2.2.2.2
proposal IKE-PROP
!
crypto ikev2 keyring IKE-KEY
peer VPN
address 0.0.0.0 0.0.0.0
pre-shared-key Passw0rd!
!
!
!
crypto ikev2 profile IKE-PROF
match fvrf any
match identity remote any
authentication remote pre-share
authentication local pre-share
keyring local IKE-KEY
!
!
!
crypto ipsec transform-set IPSEC-TRANS esp-aes 256 esp-sha256-hmac
mode tunnel
!
crypto ipsec profile IPSEC-PROF
set transform-set IPSEC-TRANS
set ikev2-profile IKE-PROF
!
!
!
!
!
!
!
interface Loopback0
ip address 1.1.1.1 255.255.255.255
!
interface Tunnel100
vrf forwarding herningco
ip address 10.0.0.2 255.255.255.0
no ip redirects
ip nhrp authentication Passw0rd
ip nhrp network-id 100
ip nhrp nhs 10.0.0.1 nbma 10.10.10.10 multicast
ip ospf network broadcast
ip ospf priority 0
tunnel source Loopback0
tunnel mode gre multipoint
tunnel key 100
tunnel protection ipsec profile IPSEC-PROF
!
interface GigabitEthernet0/0
ip address 80.200.200.6 255.255.255.252
duplex auto
speed auto
media-type rj45
!
interface GigabitEthernet0/1
vrf forwarding herningco
ip address 172.16.32.5 255.255.255.248
duplex auto
speed auto
media-type rj45
!
interface GigabitEthernet0/2
vrf forwarding herningco
ip address 172.16.33.5 255.255.255.248
duplex auto
speed auto
media-type rj45
!
interface GigabitEthernet0/3
no ip address
shutdown
duplex auto
speed auto
media-type rj45
!
router ospf 10 vrf herningco
network 10.0.0.0 0.0.0.255 area 0
network 172.16.32.0 0.0.0.7 area 0
network 172.16.33.0 0.0.0.7 area 0
!
router bgp 65001
bgp log-neighbor-changes
network 1.1.1.1 mask 255.255.255.255
neighbor 80.200.200.5 remote-as 8654
neighbor 80.200.200.5 password Passw0rd!
!
ip forward-protocol nd
!
!
no ip http server
no ip http secure-server
ip ssh version 2
!
ipv6 ioam timestamp
!
!
!
control-plane
!
banner exec ^C
**************************************************************************
* IOSv is strictly limited to use for evaluation, demonstration and IOS  *
* education. IOSv is provided as-is and is not supported by Cisco's      *
* Technical Advisory Center. Any use or disclosure, in whole or in part, *
* of the IOSv Software or Documentation to any third party for any       *
* purposes is expressly prohibited except as otherwise authorized by     *
* Cisco in writing.                                                      *
**************************************************************************^C
banner incoming ^C
**************************************************************************
* IOSv is strictly limited to use for evaluation, demonstration and IOS  *
* education. IOSv is provided as-is and is not supported by Cisco's      *
* Technical Advisory Center. Any use or disclosure, in whole or in part, *
* of the IOSv Software or Documentation to any third party for any       *
* purposes is expressly prohibited except as otherwise authorized by     *
* Cisco in writing.                                                      *
**************************************************************************^C
banner login ^C
**************************************************************************
* IOSv is strictly limited to use for evaluation, demonstration and IOS  *
* education. IOSv is provided as-is and is not supported by Cisco's      *
* Technical Advisory Center. Any use or disclosure, in whole or in part, *
* of the IOSv Software or Documentation to any third party for any       *
* purposes is expressly prohibited except as otherwise authorized by     *
* Cisco in writing.                                                      *
**************************************************************************^C
!
line con 0
exec-timeout 0 0
logging synchronous
line aux 0
line vty 0 4
exec-timeout 0 0
login local
transport input ssh
line vty 5 15
login local
transport input ssh
!
no scheduler allocate
!
end
  
  ```
</details>

<details>
<summary>CPE-02</summary>
    
  ```
  Building configuration...


Current configuration : 4753 bytes
!
! Last configuration change at 14:45:55 UTC Tue Jun 17 2025
!
version 15.9
service timestamps debug datetime msec
service timestamps log datetime msec
no service password-encryption
!
hostname CPE-02
!
boot-start-marker
boot-end-marker
!
!
vrf definition herningco
rd 65001:100
!
address-family ipv4
exit-address-family
!
no logging console
!
no aaa new-model
!
!
!
mmi polling-interval 60
no mmi auto-configure
no mmi pvc
mmi snmp-timeout 180
!
!
!
!
!
!
!
!
!
!
!
no ip domain lookup
ip domain name isp.denmark
ip cef
no ipv6 cef
!
multilink bundle-name authenticated
!
!
!
!
username Administrator privilege 15 secret 9 $9$PsqWUJaY6R4szP$EtDd.g6KLzyUDjFSQlImq6O1rx2LSFl7P.GjXc13DOI
!
redundancy
!
!
!
!
crypto ikev2 proposal IKE-PROP
encryption aes-cbc-256
integrity sha256
group 14
!
crypto ikev2 policy IKE-POL
match fvrf any
match address local 2.2.2.2
proposal IKE-PROP
!
crypto ikev2 keyring IKE-KEY
peer VPN
address 0.0.0.0 0.0.0.0
pre-shared-key Passw0rd!
!
!
!
crypto ikev2 profile IKE-PROF
match fvrf any
match identity remote any
authentication remote pre-share
authentication local pre-share
keyring local IKE-KEY
!
!
!
crypto ipsec transform-set IPSEC-TRANS esp-aes 256 esp-sha256-hmac
mode tunnel
!
crypto ipsec profile IPSEC-PROF
set transform-set IPSEC-TRANS
set ikev2-profile IKE-PROF
!
!
!
!
!
!
!
interface Loopback0
ip address 2.2.2.2 255.255.255.255
!
interface Tunnel100
vrf forwarding herningco
ip address 10.0.0.3 255.255.255.0
no ip redirects
ip nhrp authentication Passw0rd
ip nhrp network-id 100
ip nhrp nhs 10.0.0.1 nbma 10.10.10.10 multicast
ip ospf network broadcast
ip ospf priority 0
tunnel source Loopback0
tunnel mode gre multipoint
tunnel key 100
tunnel protection ipsec profile IPSEC-PROF
!
interface GigabitEthernet0/0
ip address 80.200.200.10 255.255.255.252
duplex auto
speed auto
media-type rj45
!
interface GigabitEthernet0/1
vrf forwarding herningco
ip address 172.16.33.6 255.255.255.248
duplex auto
speed auto
media-type rj45
!
interface GigabitEthernet0/2
vrf forwarding herningco
ip address 172.16.32.6 255.255.255.248
duplex auto
speed auto
media-type rj45
!
interface GigabitEthernet0/3
no ip address
shutdown
duplex auto
speed auto
media-type rj45
!
router ospf 10 vrf herningco
network 10.0.0.0 0.0.0.255 area 0
network 172.16.32.0 0.0.0.7 area 0
network 172.16.33.0 0.0.0.7 area 0
!
router bgp 65001
bgp log-neighbor-changes
network 2.2.2.2 mask 255.255.255.255
neighbor 80.200.200.9 remote-as 8654
neighbor 80.200.200.9 password Passw0rd!
!
ip forward-protocol nd
!
!
no ip http server
no ip http secure-server
ip ssh version 2
!
ipv6 ioam timestamp
!
!
!
control-plane
!
banner exec ^C
**************************************************************************
* IOSv is strictly limited to use for evaluation, demonstration and IOS  *
* education. IOSv is provided as-is and is not supported by Cisco's      *
* Technical Advisory Center. Any use or disclosure, in whole or in part, *
* of the IOSv Software or Documentation to any third party for any       *
* purposes is expressly prohibited except as otherwise authorized by     *
* Cisco in writing.                                                      *
**************************************************************************^C
banner incoming ^C
**************************************************************************
* IOSv is strictly limited to use for evaluation, demonstration and IOS  *
* education. IOSv is provided as-is and is not supported by Cisco's      *
* Technical Advisory Center. Any use or disclosure, in whole or in part, *
* of the IOSv Software or Documentation to any third party for any       *
* purposes is expressly prohibited except as otherwise authorized by     *
* Cisco in writing.                                                      *
**************************************************************************^C
banner login ^C
**************************************************************************
* IOSv is strictly limited to use for evaluation, demonstration and IOS  *
* education. IOSv is provided as-is and is not supported by Cisco's      *
* Technical Advisory Center. Any use or disclosure, in whole or in part, *
* of the IOSv Software or Documentation to any third party for any       *
* purposes is expressly prohibited except as otherwise authorized by     *
* Cisco in writing.                                                      *
**************************************************************************^C
!
line con 0
exec-timeout 0 0
logging synchronous
line aux 0
line vty 0 4
exec-timeout 0 0
login local
transport input ssh
line vty 5 15
login local
transport input ssh
!
no scheduler allocate
!
end
  
  ```
</details>

<details>
<summary>CPE-04</summary>
    
  ```
  
  
  ```
</details>

<details>
<summary>CPE-05</summary>
    
  ```
  
  
  ```
</details>

<details>
<summary>DSW1</summary>
    
  ```
  Building configuration...

Current configuration : 3988 bytes
!
! Last configuration change at 14:45:55 UTC Tue Jun 17 2025
!
version 15.2
service timestamps debug datetime msec
service timestamps log datetime msec
no service password-encryption
service compress-config
!
hostname DSW1
!
boot-start-marker
boot-end-marker
!
!
no logging console
!
username Administrator privilege 15 secret 5 $1$N4jW$2ORQVKu.Wykbcjz.nNzSD/
no aaa new-model
!
!
!
!
!
!
!
!
ip domain-name isp.denmark
ip cef
no ipv6 cef
!
!
!
spanning-tree mode mst
spanning-tree extend system-id
!
spanning-tree mst configuration
name HQ-REGION
revision 1
instance 1 vlan 100
instance 2 vlan 10
!
spanning-tree mst 1 priority 24576
spanning-tree mst 2 priority 28672
!
!
!
!
!
!
!
!
!
!
!
!
!
!
!
interface GigabitEthernet0/0
switchport access vlan 200
switchport mode access
negotiation auto
!
interface GigabitEthernet0/1
switchport access vlan 200
switchport mode access
negotiation auto
!
interface GigabitEthernet0/2
switchport trunk allowed vlan 10,99,100
switchport trunk encapsulation dot1q
switchport trunk native vlan 99
switchport mode trunk
negotiation auto
!
interface GigabitEthernet0/3
switchport trunk allowed vlan 10,99,100
switchport trunk encapsulation dot1q
switchport trunk native vlan 99
switchport mode trunk
negotiation auto
!
interface Vlan10
ip address 172.16.10.252 255.255.255.0
standby 10 ip 172.16.10.254
standby 10 priority 150
standby 10 preempt
standby 10 authentication md5 key-string Passw0rd!
!
interface Vlan100
ip address 172.16.100.252 255.255.255.0
standby 100 ip 172.16.100.254
standby 100 priority 150
standby 100 preempt
standby 100 authentication md5 key-string Passw0rd!
!
interface Vlan200
ip address 172.16.32.1 255.255.255.248
!
router ospf 10
network 172.16.10.0 0.0.0.255 area 0
network 172.16.32.0 0.0.0.7 area 0
network 172.16.100.0 0.0.0.255 area 0
!
ip forward-protocol nd
!
ip http server
ip http secure-server
!
ip ssh version 2
ip ssh server algorithm encryption aes128-ctr aes192-ctr aes256-ctr
ip ssh client algorithm encryption aes128-ctr aes192-ctr aes256-ctr
!
!
!
!
!
!
control-plane
!
banner exec ^C
IOSv - Cisco Systems Confidential -

Supplemental End User License Restrictions

This IOSv software is provided AS-IS without warranty of any kind. Under no circumstances may this software be used separate from the Cisco Modeling Labs Software that this software was provided with, or deployed or used as part of a production environment.

By using the software, you agree to abide by the terms and conditions of the Cisco End User License Agreement at http://www.cisco.com/go/eula. Unauthorized use or distribution of this software is expressly prohibited.
^C
banner incoming ^C
IOSv - Cisco Systems Confidential -

Supplemental End User License Restrictions

This IOSv software is provided AS-IS without warranty of any kind. Under no circumstances may this software be used separate from the Cisco Modeling Labs Software that this software was provided with, or deployed or used as part of a production environment.

By using the software, you agree to abide by the terms and conditions of the Cisco End User License Agreement at http://www.cisco.com/go/eula. Unauthorized use or distribution of this software is expressly prohibited.
^C
banner login ^C
IOSv - Cisco Systems Confidential -

Supplemental End User License Restrictions

This IOSv software is provided AS-IS without warranty of any kind. Under no circumstances may this software be used separate from the Cisco Modeling Labs Software that this software was provided with, or deployed or used as part of a production environment.

By using the software, you agree to abide by the terms and conditions of the Cisco End User License Agreement at http://www.cisco.com/go/eula. Unauthorized use or distribution of this software is expressly prohibited.
^C
!
line con 0
exec-timeout 0 0
line aux 0
line vty 0 4
exec-timeout 0 0
login local
transport input ssh
line vty 5 15
login local
transport input ssh
!
!
end
  
  ```
</details>

<details>
<summary>DSW2</summary>
  
  ```
  Building configuration...

    Current configuration : 3896 bytes
    !
    ! Last configuration change at 14:46:05 UTC Tue Jun 17 2025
    !
    version 15.2
    service timestamps debug datetime msec
    service timestamps log datetime msec
    no service password-encryption
    service compress-config
    !
    hostname DSW2
    !
    boot-start-marker
    boot-end-marker
    !
    !
    no logging console
    !
    username Administrator privilege 15 secret 5 $1$NDfc$MHj2E9sgWO1YcZvKmY38k/
    no aaa new-model
    !
    !
    !
    !
    !
    !
    !
    !
    ip domain-name isp.denmark
    ip cef
    no ipv6 cef
    !
    !
    !
    spanning-tree mode mst
    spanning-tree extend system-id
    !
    spanning-tree mst configuration
    name HQ-REGION
    revision 1
    instance 1 vlan 100
    instance 2 vlan 10
    !
    spanning-tree mst 1 priority 28672
    spanning-tree mst 2 priority 24576
    !
    !
    !
    !
    !
    !
    !
    !
    !
    !
    !
    !
    !
    !
    !
    interface GigabitEthernet0/0
    switchport access vlan 201
    switchport mode access
    negotiation auto
    !
    interface GigabitEthernet0/1
    switchport access vlan 201
    switchport mode access
    negotiation auto
    !
    interface GigabitEthernet0/2
    switchport trunk allowed vlan 10,99,100
    switchport trunk encapsulation dot1q
    switchport trunk native vlan 99
    switchport mode trunk
    negotiation auto
    !
    interface GigabitEthernet0/3
    switchport trunk allowed vlan 10,99,100
    switchport trunk encapsulation dot1q
    switchport trunk native vlan 99
    switchport mode trunk
    negotiation auto
    !
    interface Vlan10
    ip address 172.16.10.253 255.255.255.0
    standby 10 ip 172.16.10.254
    standby 10 authentication md5 key-string Passw0rd!
    !
    interface Vlan100
    ip address 172.16.100.253 255.255.255.0
    standby 100 ip 172.16.100.254
    standby 100 authentication md5 key-string Passw0rd!
    !
    interface Vlan201
    ip address 172.16.33.1 255.255.255.248
    !
    router ospf 10
    network 172.16.10.0 0.0.0.255 area 0
    network 172.16.33.0 0.0.0.7 area 0
    network 172.16.100.0 0.0.0.255 area 0
    !
    ip forward-protocol nd
    !
    ip http server
    ip http secure-server
    !
    ip ssh version 2
    ip ssh server algorithm encryption aes128-ctr aes192-ctr aes256-ctr
    ip ssh client algorithm encryption aes128-ctr aes192-ctr aes256-ctr
    !
    !
    !
    !
    !
    !
    control-plane
    !
    banner exec ^C
    IOSv - Cisco Systems Confidential -

    Supplemental End User License Restrictions

    This IOSv software is provided AS-IS without warranty of any kind. Under no circumstances may this software be used separate from the Cisco Modeling Labs Software that this software was provided with, or deployed or used as part of a production environment.

    By using the software, you agree to abide by the terms and conditions of the Cisco End User License Agreement at http://www.cisco.com/go/eula. Unauthorized use or distribution of this software is expressly prohibited.
    ^C
    banner incoming ^C
    IOSv - Cisco Systems Confidential -

    Supplemental End User License Restrictions

    This IOSv software is provided AS-IS without warranty of any kind. Under no circumstances may this software be used separate from the Cisco Modeling Labs Software that this software was provided with, or deployed or used as part of a production environment.

    By using the software, you agree to abide by the terms and conditions of the Cisco End User License Agreement at http://www.cisco.com/go/eula. Unauthorized use or distribution of this software is expressly prohibited.
    ^C
    banner login ^C
    IOSv - Cisco Systems Confidential -

    Supplemental End User License Restrictions

    This IOSv software is provided AS-IS without warranty of any kind. Under no circumstances may this software be used separate from the Cisco Modeling Labs Software that this software was provided with, or deployed or used as part of a production environment.

    By using the software, you agree to abide by the terms and conditions of the Cisco End User License Agreement at http://www.cisco.com/go/eula. Unauthorized use or distribution of this software is expressly prohibited.
    ^C
    !
    line con 0
    exec-timeout 0 0
    line aux 0
    line vty 0 4
    exec-timeout 0 0
    login local
    transport input ssh
    line vty 5 15
    login local
    transport input ssh
    !
    !
end
  
  ```
</details>

<details>
<summary>ASW1</summary>
    
  ```
  Building configuration...

          Current configuration : 3807 bytes
          !
          ! Last configuration change at 14:46:06 UTC Tue Jun 17 2025
          !
          version 15.2
          service timestamps debug datetime msec
          service timestamps log datetime msec
          no service password-encryption
          service compress-config
          !
          hostname ASW1
          !
          boot-start-marker
          boot-end-marker
          !
          !
          no logging console
          !
          username Administrator password 0 Passw0rd!
          no aaa new-model
          !
          !
          !
          !
          !
          ip arp inspection vlan 10,100
          ip arp inspection filter ACL vlan  100
          !
          !
          !
          ip domain-name isp.denmark
          ip cef
          no ipv6 cef
          !
          !
          !
          spanning-tree mode mst
          spanning-tree portfast edge default
          spanning-tree extend system-id
          !
          spanning-tree mst configuration
           name HQ-REGION
           revision 1
           instance 1 vlan 100
           instance 2 vlan 10
          !
          !
          !
          !
          !
          !
          !
          !
          !
          !
          !
          !
          !
          !
          !
          !
          interface GigabitEthernet0/0
           switchport trunk allowed vlan 10,99,100
           switchport trunk encapsulation dot1q
           switchport trunk native vlan 99
           switchport mode trunk
           ip arp inspection trust
           negotiation auto
           spanning-tree portfast disable
          !
          interface GigabitEthernet0/1
           switchport trunk allowed vlan 10,99,100
           switchport trunk encapsulation dot1q
           switchport trunk native vlan 99
           switchport mode trunk
           ip arp inspection trust
           negotiation auto
           spanning-tree portfast disable
          !
          interface GigabitEthernet0/2
           switchport access vlan 100
           switchport mode access
           switchport port-security violation restrict
           switchport port-security
           negotiation auto
           spanning-tree bpduguard enable
          !
          interface GigabitEthernet0/3
           switchport access vlan 999
           switchport mode access
           shutdown
           negotiation auto
          !
          ip forward-protocol nd
          !
          ip http server
          ip http secure-server
          !
          ip ssh version 2
          ip ssh server algorithm encryption aes128-ctr aes192-ctr aes256-ctr
          ip ssh client algorithm encryption aes128-ctr aes192-ctr aes256-ctr
          !
          !
          ip source binding 5254.000E.9E83 vlan 100 172.16.100.100 interface Gi0/2
          !
          arp access-list ACL
           permit ip host 172.16.100.100 mac any
          !
          !
          !
          !
          control-plane
          !
          banner exec ^C
          IOSv - Cisco Systems Confidential -

          Supplemental End User License Restrictions

          This IOSv software is provided AS-IS without warranty of any kind. Under no circumstances may this software be used separate from the Cisco Modeling Labs Software that this software was provided with, or deployed or used as part of a production environment.

          By using the software, you agree to abide by the terms and conditions of the Cisco End User License Agreement at http://www.cisco.com/go/eula. Unauthorized use or distribution of this software is expressly prohibited.
          ^C
          banner incoming ^C
          IOSv - Cisco Systems Confidential -

          Supplemental End User License Restrictions

          This IOSv software is provided AS-IS without warranty of any kind. Under no circumstances may this software be used separate from the Cisco Modeling Labs Software that this software was provided with, or deployed or used as part of a production environment.

          By using the software, you agree to abide by the terms and conditions of the Cisco End User License Agreement at http://www.cisco.com/go/eula. Unauthorized use or distribution of this software is expressly prohibited.
          ^C
          banner login ^C
          IOSv - Cisco Systems Confidential -

          Supplemental End User License Restrictions

          This IOSv software is provided AS-IS without warranty of any kind. Under no circumstances may this software be used separate from the Cisco Modeling Labs Software that this software was provided with, or deployed or used as part of a production environment.

          By using the software, you agree to abide by the terms and conditions of the Cisco End User License Agreement at http://www.cisco.com/go/eula. Unauthorized use or distribution of this software is expressly prohibited.
          ^C
          !
          line con 0
           exec-timeout 0 0
           logging synchronous
          line aux 0
          line vty 0 4
           exec-timeout 0 0
           login local
           transport input ssh
          line vty 5 15
           login local
           transport input ssh
          !
          !
          end
  
  ```
</details>

<details>
<summary>ASW2</summary>
    
  ```
  Building configuration...

Current configuration : 3710 bytes
!
! Last configuration change at 14:46:08 UTC Tue Jun 17 2025
!
version 15.2
service timestamps debug datetime msec
service timestamps log datetime msec
no service password-encryption
service compress-config
!
hostname ASW2
!
boot-start-marker
boot-end-marker
!
!
no logging console
!
username Administrator password 0 Passw0rd!
no aaa new-model
!
!
!
!
!
ip arp inspection vlan 10,100
ip arp inspection filter ACL vlan  10
!
!
!
ip domain-name isp.denmark
ip cef
no ipv6 cef
!
!
!
spanning-tree mode mst
spanning-tree portfast edge default
spanning-tree extend system-id
!
spanning-tree mst configuration
name HQ-REGION
revision 1
instance 1 vlan 100
instance 2 vlan 10
!
!
!
!
!
!
!
!
!
!
!
!
!
!
!
!
interface GigabitEthernet0/0
switchport trunk allowed vlan 10,99,100
switchport trunk encapsulation dot1q
switchport trunk native vlan 99
switchport mode trunk
ip arp inspection trust
negotiation auto
spanning-tree portfast disable
!
interface GigabitEthernet0/1
switchport trunk allowed vlan 10,99,100
switchport trunk encapsulation dot1q
switchport trunk native vlan 99
switchport mode trunk
ip arp inspection trust
negotiation auto
spanning-tree portfast disable
!
interface GigabitEthernet0/2
switchport access vlan 10
switchport mode access
switchport port-security violation restrict
switchport port-security
negotiation auto
spanning-tree bpduguard enable
!
interface GigabitEthernet0/3
switchport access vlan 999
switchport mode access
shutdown
negotiation auto
!
ip forward-protocol nd
!
ip http server
ip http secure-server
!
ip ssh version 2
ip ssh server algorithm encryption aes128-ctr aes192-ctr aes256-ctr
ip ssh client algorithm encryption aes128-ctr aes192-ctr aes256-ctr
!
!
!
arp access-list ACL
permit ip host 172.16.10.100 mac any
!
!
!
!
control-plane
!
banner exec ^C
IOSv - Cisco Systems Confidential -

Supplemental End User License Restrictions

This IOSv software is provided AS-IS without warranty of any kind. Under no circumstances may this software be used separate from the Cisco Modeling Labs Software that this software was provided with, or deployed or used as part of a production environment.

By using the software, you agree to abide by the terms and conditions of the Cisco End User License Agreement at http://www.cisco.com/go/eula. Unauthorized use or distribution of this software is expressly prohibited.
^C
banner incoming ^C
IOSv - Cisco Systems Confidential -

Supplemental End User License Restrictions

This IOSv software is provided AS-IS without warranty of any kind. Under no circumstances may this software be used separate from the Cisco Modeling Labs Software that this software was provided with, or deployed or used as part of a production environment.

By using the software, you agree to abide by the terms and conditions of the Cisco End User License Agreement at http://www.cisco.com/go/eula. Unauthorized use or distribution of this software is expressly prohibited.
^C
banner login ^C
IOSv - Cisco Systems Confidential -

Supplemental End User License Restrictions

This IOSv software is provided AS-IS without warranty of any kind. Under no circumstances may this software be used separate from the Cisco Modeling Labs Software that this software was provided with, or deployed or used as part of a production environment.

By using the software, you agree to abide by the terms and conditions of the Cisco End User License Agreement at http://www.cisco.com/go/eula. Unauthorized use or distribution of this software is expressly prohibited.
^C
!
line con 0
exec-timeout 0 0
line aux 0
line vty 0 4
exec-timeout 0 0
login local
transport input ssh
line vty 5 15
login local
transport input ssh
!
!
end
  
  ```
</details>

<details>
<summary>BR1-SW</summary>
    
  ```
  Building configuration...

Current configuration : 2875 bytes
!
! Last configuration change at 14:46:08 UTC Tue Jun 17 2025
!
version 15.2
service timestamps debug datetime msec
service timestamps log datetime msec
no service password-encryption
service compress-config
!
hostname BR1-SW
!
boot-start-marker
boot-end-marker
!
!
no logging console
!
username Administrator privilege 15 secret 5 $1$G19y$jsDcYPAqWDSGxd1sQprvc.
no aaa new-model
!
!
!
!
!
!
!
!
ip domain-name isp.denmark
ip cef
no ipv6 cef
!
!
!
spanning-tree mode pvst
spanning-tree extend system-id
!
!
!
!
!
!
!
!
!
!
!
!
!
!
!
interface GigabitEthernet0/0
negotiation auto
!
interface GigabitEthernet0/1
negotiation auto
!
interface GigabitEthernet0/2
negotiation auto
!
interface GigabitEthernet0/3
negotiation auto
!
ip forward-protocol nd
!
ip http server
ip http secure-server
!
ip ssh version 2
ip ssh server algorithm encryption aes128-ctr aes192-ctr aes256-ctr
ip ssh client algorithm encryption aes128-ctr aes192-ctr aes256-ctr
!
!
!
!
!
!
control-plane
!
banner exec ^C
IOSv - Cisco Systems Confidential -

Supplemental End User License Restrictions

This IOSv software is provided AS-IS without warranty of any kind. Under no circumstances may this software be used separate from the Cisco Modeling Labs Software that this software was provided with, or deployed or used as part of a production environment.

By using the software, you agree to abide by the terms and conditions of the Cisco End User License Agreement at http://www.cisco.com/go/eula. Unauthorized use or distribution of this software is expressly prohibited.
^C
banner incoming ^C
IOSv - Cisco Systems Confidential -

Supplemental End User License Restrictions

This IOSv software is provided AS-IS without warranty of any kind. Under no circumstances may this software be used separate from the Cisco Modeling Labs Software that this software was provided with, or deployed or used as part of a production environment.

By using the software, you agree to abide by the terms and conditions of the Cisco End User License Agreement at http://www.cisco.com/go/eula. Unauthorized use or distribution of this software is expressly prohibited.
^C
banner login ^C
IOSv - Cisco Systems Confidential -

Supplemental End User License Restrictions

This IOSv software is provided AS-IS without warranty of any kind. Under no circumstances may this software be used separate from the Cisco Modeling Labs Software that this software was provided with, or deployed or used as part of a production environment.

By using the software, you agree to abide by the terms and conditions of the Cisco End User License Agreement at http://www.cisco.com/go/eula. Unauthorized use or distribution of this software is expressly prohibited.
^C
!
line con 0
exec-timeout 0 0
line aux 0
line vty 0 4
exec-timeout 0 0
login local
transport input ssh
line vty 5 15
login local
transport input ssh
!
!
end
  
  ```
</details>

<details>
<summary>BR2-SW</summary>
    
  ```
  Building configuration...

Current configuration : 2875 bytes
!
! Last configuration change at 14:46:09 UTC Tue Jun 17 2025
!
version 15.2
service timestamps debug datetime msec
service timestamps log datetime msec
no service password-encryption
service compress-config
!
hostname BR2-SW
!
boot-start-marker
boot-end-marker
!
!
no logging console
!
username Administrator privilege 15 secret 5 $1$vytC$S3ir69587s2xxn0UXS8EJ/
no aaa new-model
!
!
!
!
!
!
!
!
ip domain-name isp.denmark
ip cef
no ipv6 cef
!
!
!
spanning-tree mode pvst
spanning-tree extend system-id
!
!
!
!
!
!
!
!
!
!
!
!
!
!
!
interface GigabitEthernet0/0
negotiation auto
!
interface GigabitEthernet0/1
negotiation auto
!
interface GigabitEthernet0/2
negotiation auto
!
interface GigabitEthernet0/3
negotiation auto
!
ip forward-protocol nd
!
ip http server
ip http secure-server
!
ip ssh version 2
ip ssh server algorithm encryption aes128-ctr aes192-ctr aes256-ctr
ip ssh client algorithm encryption aes128-ctr aes192-ctr aes256-ctr
!
!
!
!
!
!
control-plane
!
banner exec ^C
IOSv - Cisco Systems Confidential -

Supplemental End User License Restrictions

This IOSv software is provided AS-IS without warranty of any kind. Under no circumstances may this software be used separate from the Cisco Modeling Labs Software that this software was provided with, or deployed or used as part of a production environment.

By using the software, you agree to abide by the terms and conditions of the Cisco End User License Agreement at http://www.cisco.com/go/eula. Unauthorized use or distribution of this software is expressly prohibited.
^C
banner incoming ^C
IOSv - Cisco Systems Confidential -

Supplemental End User License Restrictions

This IOSv software is provided AS-IS without warranty of any kind. Under no circumstances may this software be used separate from the Cisco Modeling Labs Software that this software was provided with, or deployed or used as part of a production environment.

By using the software, you agree to abide by the terms and conditions of the Cisco End User License Agreement at http://www.cisco.com/go/eula. Unauthorized use or distribution of this software is expressly prohibited.
^C
banner login ^C
IOSv - Cisco Systems Confidential -

Supplemental End User License Restrictions

This IOSv software is provided AS-IS without warranty of any kind. Under no circumstances may this software be used separate from the Cisco Modeling Labs Software that this software was provided with, or deployed or used as part of a production environment.

By using the software, you agree to abide by the terms and conditions of the Cisco End User License Agreement at http://www.cisco.com/go/eula. Unauthorized use or distribution of this software is expressly prohibited.
^C
!
line con 0
exec-timeout 0 0
line aux 0
line vty 0 4
exec-timeout 0 0
login local
transport input ssh
line vty 5 15
login local
transport input ssh
!
!
end
  
  ```
</details>