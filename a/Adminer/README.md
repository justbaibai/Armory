[+] Credits: John Page (aka hyp3rlinx)		
[+] Website: hyp3rlinx.altervista.org
[+] Source:  http://hyp3rlinx.altervista.org/advisories/ADMINER-UNAUTHENTICATED-SERVER-SIDE-REQUEST-FORGERY.txt
[+] ISR: apparition security           
 


Vendor:
==============
www.adminer.org


Product:
================
Adminer <= v4.3.1 

Adminer (formerly phpMinAdmin) is a full-featured database management tool written in PHP. Conversely to phpMyAdmin, it consist of a
single file ready to deploy to the target server. Adminer is available for MySQL, PostgreSQL, SQLite, MS SQL, Oracle, Firebird, SimpleDB, Elasticsearch and MongoDB.

https://github.com/vrana/adminer/releases/


Vulnerability Type:
===================
Server Side Request Forgery


CVE Reference:
==============
N/A


Security Issue:
================
Adminer allows unauthenticated connections to be initiated to arbitrary systems/ports. This vulnerability can be used to potentially bypass firewalls to
identify internal hosts and perform port scanning of other servers for reconnaissance purposes. Funny thing is Adminer throttles invalid login attempts
but allows endless unauthorized HTTP connections to other systems as long as your not trying to authenticate to Adminer itself.

Situations where Adminer can talk to a server that we are not allowed to (ACL) and where we can talk to the server hosting Adminer, it can do recon for us.

Recently in LAN I was firewalled off from a server, however another server running Adminer I can talk to. Also, that Adminer server can talk to the target.
Since Adminer suffers from Server-Side Request Forgery, I can scan for open ports and gather information from that firewalled off protected server.
This allowed me to not only bypass the ACL but also hide from the threat detection system (IDS) monitoring east west connections. 

However, sysadmins who check the logs on the server hosting Adminer application will see our port scans.

root@lamp log/apache2# cat other_vhosts_access.log
localhost:12322 ATTACKER-IP - - [2/Jan/2018:14:25:11 +0000] "GET ///?server=TARGET-IP:21&username= HTTP/1.1" 403 1429 "-" "-"
localhost:12322 ATTACKER-IP - - [2/Jan/2018:14:26:24 +0000] "GET ///?server=TARGET-IP:22&username= HTTP/1.1" 403 6019 "-" "-"
localhost:12322 ATTACKER-IP - - [2/Jan/2018:14:26:56 +0000] "GET ///?server=TARGET-IP:23&username= HTTP/1.1" 403 6021 "-" "-"


Details:
==================
By comparing different failed error responses from Adminer when making SSRF bogus connections, I figured out which ports are open/closed.

Port open ==> Lost connection to MySQL server at 'reading initial communication packet
Port open ==> MySQL server has gone away
Port open ==> Bad file descriptor 
Port closed ==> Can't connect to MySQL server on '<TARGET-IP>';
Port closed ==> No connection could be made because the target machine actively refused it
Port closed ==> A connection attempt failed. 

This worked so well for me I wrote a quick port scanner 'PortMiner' as a proof of concept that leverages Adminer SSRF vulnerability.


PortMiner observations:
======================
No response 'read operation timed out' means the port is possibly open or filtered and should be given a closer look if possible. This seems to occur when scanning
Web server ports like 80, 443. However, when we get error responses like the ones above from the server we can be fairly certain a port is either open/closed. 

Quick POC:
echo -e 'HTTP/1.1 200 OK\r\n\r\n' | nc -l -p 5555
Use range 5555-5555
