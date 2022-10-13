---
author: [Felicitas Pojtinger]
date: "2022-10-12"
subject: "Hacking Report"
keywords: [hdm-stuttgart, hacking]
subtitle: "Hack the Box"
lang: en-US
---

# Uni Hacking Report (Shared)

**Machine**: Shared (10.10.11.172)

```shell
$ nmap -v -p- 10.10.11.172
Starting Nmap 7.92 ( https://nmap.org ) at 2022-10-12 12:12 CEST
Initiating Ping Scan at 12:12
Scanning 10.10.11.172 [2 ports]
Completed Ping Scan at 12:12, 0.03s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host. at 12:12
Completed Parallel DNS resolution of 1 host. at 12:12, 0.16s elapsed
Initiating Connect Scan at 12:12
Scanning 10.10.11.172 [65535 ports]
Discovered open port 22/tcp on 10.10.11.172
Discovered open port 443/tcp on 10.10.11.172
Discovered open port 80/tcp on 10.10.11.172
Increasing send delay for 10.10.11.172 from 0 to 5 due to max_successful_tryno increase to 4
```

```shell
$ curl -vvv -L http://10.10.11.172 | xclip -selection c
*   Trying 10.10.11.172:80...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0* Connected to 10.10.11.172 (10.10.11.172) port 80 (#0)
> GET / HTTP/1.1
> Host: 10.10.11.172
> User-Agent: curl/7.85.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 301 Moved Permanently
< Server: nginx/1.18.0
< Date: Wed, 12 Oct 2022 10:17:45 GMT
< Content-Type: text/html
< Content-Length: 169
< Connection: keep-alive
< Location: http://shared.htb
<
* Ignoring the response-body
{ [169 bytes data]
100   169  100   169    0     0   2799      0 --:--:-- --:--:-- --:--:--  2816
* Connection #0 to host 10.10.11.172 left intact
* Issue another request to this URL: 'http://shared.htb/'
* Could not resolve host: shared.htb
* Closing connection 1
curl: (6) Could not resolve host: shared.htb
```

```shell
$ curl -k -vvv -L https://10.10.11.172 | xclip -selection c
*   Trying 10.10.11.172:443...
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0* Connected to 10.10.11.172 (10.10.11.172) port 443 (#0)
* ALPN: offers h2
* ALPN: offers http/1.1
} [5 bytes data]
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
} [512 bytes data]
* TLSv1.3 (IN), TLS handshake, Server hello (2):
{ [122 bytes data]
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
{ [15 bytes data]
* TLSv1.3 (IN), TLS handshake, Certificate (11):
{ [914 bytes data]
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
{ [264 bytes data]
* TLSv1.3 (IN), TLS handshake, Finished (20):
{ [52 bytes data]
* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
} [1 bytes data]
* TLSv1.3 (OUT), TLS handshake, Finished (20):
} [52 bytes data]
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* ALPN: server accepted h2
* Server certificate:
*  subject: C=US; ST=None; L=None; O=HTB; CN=*.shared.htb
*  start date: Mar 20 13:37:14 2022 GMT
*  expire date: Mar 15 13:37:14 2042 GMT
*  issuer: C=US; ST=None; L=None; O=HTB; CN=*.shared.htb
*  SSL certificate verify result: self signed certificate (18), continuing anyway.
* Using HTTP2, server supports multiplexing
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
} [5 bytes data]
* h2h3 [:method: GET]
* h2h3 [:path: /]
* h2h3 [:scheme: https]
* h2h3 [:authority: 10.10.11.172]
* h2h3 [user-agent: curl/7.85.0]
* h2h3 [accept: */*]
* Using Stream ID: 1 (easy handle 0x5579c203f1e0)
} [5 bytes data]
> GET / HTTP/2
> Host: 10.10.11.172
> user-agent: curl/7.85.0
> accept: */*
>
{ [5 bytes data]
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
{ [249 bytes data]
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
{ [249 bytes data]
* old SSL session ID is stale, removing
{ [5 bytes data]
* Connection state changed (MAX_CONCURRENT_STREAMS == 128)!
} [5 bytes data]
< HTTP/2 301
< server: nginx/1.18.0
< date: Wed, 12 Oct 2022 10:18:37 GMT
< content-type: text/html
< content-length: 169
< location: https://shared.htb
<
{ [5 bytes data]
* Ignoring the response-body
{ [169 bytes data]
100   169  100   169    0     0   2014      0 --:--:-- --:--:-- --:--:--  2036
* Connection #0 to host 10.10.11.172 left intact
* Issue another request to this URL: 'https://shared.htb/'
* Could not resolve host: shared.htb
* Closing connection 1
curl: (6) Could not resolve host: shared.htb
```

https://www.cybersecurity-help.cz/vdb/SB2021052543

https://github.com/dentremor/ctf-notes/blob/main/docs/main.md

```shell
$ git clone https://github.com/danielmiessler/SecLists.git ~/Downloads
$ ffuf -w ~/Downloads/SecLists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ -u http://FUZZ.10.10.11.172/
# No results
$ ffuf -w ~/Downloads/SecLists/Discovery/DNS/bitquark-subdomains-top100000.txt:FUZZ -u http://10.10.11.172/ -H 'Host: FUZZ.shared.htb' -fs 169 -s
www
checkout

# /etc/hosts
10.10.11.172 shared.htb www.shared.htb checkout.shared.htb
```

https://shared.htb/index.php?

```shell
$ ffuf -w ~/Downloads/SecLists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://shared.htb/FUZZ

# Makefile
1 install: composer assets$
2 $
3 composer:$
4 >   composer install$
5 $
6 assets:$
7 >   ./tools/assets/build.sh$

$ ffuf -w ~/Downloads/SecLists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://shared.htb/FUZZ -fs 0,169
api                     [Status: 401, Size: 16, Words: 2, Lines: 1, Duration: 2383ms]
Makefile                [Status: 200, Size: 88, Words: 4, Lines: 8, Duration: 21ms]
apis                    [Status: 401, Size: 16, Words: 2, Lines: 1, Duration: 1120ms]
apidocs                 [Status: 401, Size: 16, Words: 2, Lines: 1, Duration: 778ms]
apilist                 [Status: 401, Size: 16, Words: 2, Lines: 1, Duration: 828ms]
```

exploit/linux/http/php_imap_open_rce does not work

Set cookie `custom_cart` to `{"YCS98E4A":"'"}`: No result
