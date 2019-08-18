---
layout: post
title: "IPv6 Local Neighbor Discovery Using Router Advertisement"
date: 2019-08-18
categories:
    - blog
keywords: "ipv6, network scanner, router advertisement"
---

Back in March 2016, i had decided to learn more about IPv6 and to do so I came out with an idea to build a network scanner specifically for IPv6 network which can be found [here](https://github.com/fakhrizulkifli/6Map){:target="_blank"} and of course, it is unfinished!. During the development, I relied heavily on rfc4861 specification to utilize the Neighbor Solicitation to discover hosts on link-local.

So the idea was to send a Router Advertisement message to the multicast address for all available IPv6 devices within the same network segment. Further reading and googling left me a mailing list thread which discussed some issues in a Metasploit's auxiliary [scanner](https://www.rapid7.com/db/modules/auxiliary/scanner/discovery/ipv6_neighbor_router_advertisement){:target="_blank"}. Every time the scanner scans the entire network, it leaves the hosts in a denial-of-service state to the IPv6 network due to the misconfigured routing table. Turned out the scanner was sending the RA message with a lifetime of 1800 seconds. So within this timeframe, the hosts will not be able to have proper IPv6 connection until its the RA source address is flushed away from the routing table. So i created a patch for the scanner and set the RA lifetime to 0 indicating it is not a default router and thus not going to be appeared in the host's routing table.

Patch diff:
```
From b1e9f44ca26072ba9594ef7d034e47283f220b2f Mon Sep 17 00:00:00 2001
From: Fakhri Zulkifli <d0lph1n98@yahoo.com>
Date: Sun, 6 Mar 2016 03:23:37 +0800
Subject: [PATCH] IPv6 Neighbor Advertisement Enhancement

http://seclists.org/nmap-dev/2011/q2/79

1. Shorten router advertisement payload lifetime.
2. Randomize address prefix.
3. Prevent from getting into default router list.
---
 .../ipv6_neighbor_router_advertisement.rb     | 25 +++++++++++--------
 1 file changed, 14 insertions(+), 11 deletions(-)

diff --git a/modules/auxiliary/scanner/discovery/ipv6_neighbor_router_advertisement.rb b/modules/auxiliary/scanner/discovery/ipv6_neighbor_router_advertisement.rb
index 08e6967dfe1..4b1d988dce2 100644
--- a/modules/auxiliary/scanner/discovery/ipv6_neighbor_router_advertisement.rb
+++ b/modules/auxiliary/scanner/discovery/ipv6_neighbor_router_advertisement.rb
@@ -20,7 +20,7 @@ def initialize
         the host portion of the IPv6 address.  Use NDP host solicitation to
         determine if the IP address is valid'
     },
-    'Author'      => 'wuntee',
+    'Author'      => 'wuntee, d0lph1n98',
     'License'     => MSF_LICENSE,
     'References'    =>
     [
@@ -33,20 +33,22 @@ def initialize
       OptInt.new('TIMEOUT_NEIGHBOR', [true, "Time (seconds) to listen for a solicitation response.", 1])
     ], self.class)
 
-    register_advanced_options(
-      [
-        OptString.new('PREFIX', [true, "Prefix that each host should get an IPv6 address from",
-          "2001:1234:DEAD:BEEF::"]
-        )
-      ], self.class)
-
     deregister_options('SNAPLEN', 'FILTER', 'RHOST', 'PCAPFILE')
   end
 
+  def generate_prefix()
+    max = 16 ** 4
+    prefix = "2001::"
+    (0..2).each do
+        prefix << "%x::" % Random.rand(0..max)
+    end
+    return prefix
+  end
+
   def listen_for_neighbor_solicitation(opts = {})
     hosts = []
     timeout = opts['TIMEOUT'] || datastore['TIMEOUT']
-    prefix = opts['PREFIX'] || datastore['PREFIX']
+    prefix = @prefix
 
     max_epoch = ::Time.now.to_i + timeout
     autoconf_prefix = IPAddr.new(prefix).to_string().slice(0..19)
@@ -94,7 +96,7 @@ def create_router_advertisment(opts={})
     smac = @smac
     shost = opts['SHOST'] || datastore['SHOST'] || ipv6_link_address
     lifetime = opts['LIFETIME'] || datastore['TIMEOUT']
-    prefix = opts['PREFIX'] || datastore['PREFIX']
+    prefix = @prefix
     plen = 64
     dmac = "33:33:00:00:00:01"
 
@@ -141,7 +143,7 @@ def router_advertisement_payload
     checksum = 0
     hop_limit = 0
     flags = 0x08
-    lifetime = 1800
+    lifetime = 0
     reachable = 0
     retrans = 0
     [type, code, checksum, hop_limit, flags,
@@ -152,6 +154,7 @@ def run
     # Start capture
     open_pcap({'FILTER' => "icmp6"})
 
+    @prefix = generate_prefix()
     @netifaces = true
     if not netifaces_implemented?
       print_error("WARNING : Pcaprub is not uptodate, some functionality will not be available")

```

---
References:
1. https://github.com/fakhrizulkifli/6Map
2. https://www.rapid7.com/db/modules/auxiliary/scanner/discovery/ipv6_neighbor_router_advertisement
3. https://seclists.org/nmap-dev/2011/q2/79
4. http://www.rfcreader.com/#rfc4861
5. http://www.rfcreader.com/#rfc6104