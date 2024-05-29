---
title: "深入解析 Python descriptor 與其 opcode 處理"
date: 2024-05-25T10:38:43+08:00
draft: false
ShowToc: true
---

## 環境

Python 3.9

https://github.com/python/cpython/blob/3.9/


{{< info >}}
可以發現 Python 3.10 中此處 [ceval.c](https://github.com/python/cpython/blob/516a6d4237fe3b9fb02a800b81a0a3153f78685e/Python/ceval.c#L3406C9-L3423C28) 的實作較不同 3.9，日後可研究實際是何處不同。
{{< /info >}}

## 定義
- 在 Python 中一個類別只要定義了 `__get__` 、 `__set__` 或 `__delete__` 都會將這個類別變為描述器

https://docs.python.org/3/howto/descriptor.html#definition-and-introduction

```python {linenos=true}
class Name:
    def __get__(self, obj, objtype):
        print("descriptor trigger!")
        return "meow"

class A:
    name = Name()

o = A()
print(o.name)
"""
descriptor trigger!
meow
"""
```

此程式會輸出 meow，代表說我們在使用 `o.name` 時，會去呼叫 Name 這個類別的 `__get__` 函式。

接下來，我們針對 `foo.b` 此種操作進行分析研究，便會發現描述器的身影。

## LOAD_ATTR


{{< info >}}
`LOAD_ATTR` 在 Python bytescode 中是個十分常見的 opcode，當我們使用形如 `a.b` 的操作（求某個值）時，就會使用這個 opcode，另外，若我們要在這裡保存值，就會使用 `STORE_ATTR`。
{{< /info >}}

使用 `dis` 這個模組來反組譯分析 CPython bytecode

https://docs.python.org/zh-tw/3/library/dis.html


```plain
In [2]: dis.dis("foo.b")
  1           0 LOAD_NAME                0 (foo)
              2 LOAD_ATTR                1 (b)
              4 RETURN_VALUE
```

```plain
In [3]: dis.dis("foo.b = 'hello'")
  1           0 LOAD_CONST               0 ('hello')
              2 LOAD_NAME                0 (foo)
              4 STORE_ATTR               1 (b)
              6 LOAD_CONST               1 (None)
              8 RETURN_VALUE
```

接下來，我們要理解描述器的原理，我們就必須去看這兩個 opcode 的原始碼，接下來以 `LOAD_ATTR` 為例。

可以看到在 [Python/ceval.c](https://github.com/python/cpython/blob/200762426b4b9878c33cdef620aa1a51631dd5df/Python/ceval.c#L2993) 中，包含了 `LOAD_ATTR` 的 case，又可以發現，他主要是使用了 `PyObject_GetAttr`。

```c {linenos=true}
case TARGET(LOAD_ATTR): {
    PyObject *name = GETITEM(names, oparg);
    PyObject *owner = TOP();
    PyObject *res = PyObject_GetAttr(owner, name);
    Py_DECREF(owner);
    SET_TOP(res);
    if (res == NULL)
        goto error;
    DISPATCH();
}
```

經過追蹤，發現最後的運作邏輯位於 [Objects/object.c](https://github.com/python/cpython/blob/1cf03010865c66c2c3286ffdafd55e7ce2d97444/Objects/object.c#L1542) 的 _PyObject_GenericGetAttrWithDict 中。顯然，tp 存的便是 object 的 type，接下來就著重在分析這個函式。


```c {linenos=true}
_PyObject_GenericGetAttrWithDict(PyObject *obj, PyObject *name,
                                 PyObject *dict, int suppress)
{
    /* Make sure the logic of _PyObject_GetMethod is in sync with
       this method.

       When suppress=1, this function suppresses AttributeError.
    */

    PyTypeObject *tp = Py_TYPE(obj);
    PyObject *descr = NULL;
    PyObject *res = NULL;
    descrgetfunc f;

...

    descr = _PyType_Lookup(tp, name);
```


往下看到 line17，發現他正在型別裡面尋找 name，也就代表了**我們的描述器必須要定義到型別的類別定義中，才算有效的（在 `__init__` 中無效，將不會觸發 descriptor）**

{{< info >}}
```python {linenos=true}
class A:
    name = "meow"
    def __init__(self):
        self.owo = "owo"

o = A()
tp = type(o)
print(tp.name)
print(tp.owo)
"""
meow
Traceback (most recent call last):
  File "/tmp/descriptor2.py", line 9, in <module>
    print(tp.owo)
AttributeError: type object 'A' has no attribute 'owo'
"""
```
例如上方的例子，line7 中將 o 的 型別存到 tp 變數中，line9 存取了 tp.owo 就會報錯，原因是這個變數（`owo`）在這個類別被實例化後，呼叫 `__init__` 後才會被**定義**，也就是放到實例化後的物件的 `__dict__` 中。
反之，由於 `name` 本來就是成員變數，因此便可成功存取。
{{< /info >}}


```c {linenos=true}
f = NULL;
if (descr != NULL) {
    Py_INCREF(descr);
    f = Py_TYPE(descr)->tp_descr_get;
    if (f != NULL && PyDescr_IsData(descr)) {
        res = f(descr, obj, (PyObject *)Py_TYPE(obj));
        if (res == NULL && suppress &&
                	PyErr_ExceptionMatches(PyExc_AttributeError)) {
			PyErr_Clear();
		}
		goto done;
	}
}
```

往下看，在確定取得 `descr` 後，line4 嘗試地在描述器中找 `tp_descr_get`，也就是我們剛才定義的 `__get__` 函數，注意他有將結果存下來 `f = Py_TYPE(descr)->tp_descr_get;`，後面會使用到。

line5 這個描述器拿去 `PyDescr_IsData` 判斷，這個判斷就是看他有沒有 `__set__` 函數 。

也就是當目前這個描述器同時有 `__get__` 與 `__set__` 函數時，就會立刻進行 `__get__` 內的操作！


```c {linenos=true}
    if (dict == NULL) {
        /* Inline _PyObject_GetDictPtr */
        dictoffset = tp->tp_dictoffset;
        if (dictoffset != 0) {
            if (dictoffset < 0) {
                Py_ssize_t tsize = Py_SIZE(obj);
                if (tsize < 0) {
                    tsize = -tsize;
                }
                size_t size = _PyObject_VAR_SIZE(tp, tsize);
                _PyObject_ASSERT(obj, size <= PY_SSIZE_T_MAX);

                dictoffset += (Py_ssize_t)size;
                _PyObject_ASSERT(obj, dictoffset > 0);
                _PyObject_ASSERT(obj, dictoffset % SIZEOF_VOID_P == 0);
            }
            dictptr = (PyObject **) ((char *)obj + dictoffset);
            dict = *dictptr;
        }
    }
    if (dict != NULL) {
        Py_INCREF(dict);
        res = PyDict_GetItemWithError(dict, name);
        if (res != NULL) {
            Py_INCREF(res);
            Py_DECREF(dict);
            goto done;
        }
        else {
            Py_DECREF(dict);
            if (PyErr_Occurred()) {
                if (suppress && PyErr_ExceptionMatches(PyExc_AttributeError)) {
                    PyErr_Clear();
                }
                else {
                    goto done;
                }
            }
        }
    }
```


繼續往下看到 dict 的操作，這個非常單純。line1-20就是找到這個物件本身的 `__dict__`。line21-40 則是從 `__dict__` 中看有沒有要找的 name，有的話則回傳。

{{< info >}}
⚠️注意，這邊 dict 的操作都是對原物件做事，並不是對描述器。

以 `a.b` 為例，這邊都是在 `a` 上操作。
{{< /info >}}

```c {linenos=true}
    if (f != NULL) {
        res = f(descr, obj, (PyObject *)Py_TYPE(obj));
        if (res == NULL && suppress &&
                PyErr_ExceptionMatches(PyExc_AttributeError)) {
            PyErr_Clear();
        }
        goto done;
    }
```

接下來，這個部分代表了，有 `__get__` 但沒有 `__set__` 函數時，就會在這回傳。

注意，這邊用到的 `f` 是之前在判斷同時有 `__get__` 與 `__set__` 的時候存下來的。

```c {linenos=true}
    if (descr != NULL) {
        res = descr;
        descr = NULL;
        goto done;
    }

    if (!suppress) {
        PyErr_Format(PyExc_AttributeError,
                     "'%.50s' object has no attribute '%U'",
                     tp->tp_name, name);
    }
```

最後，若有 `descr` 則直接回傳，否則就報錯，就是大家常看到的 `type object 'A' has no attribute 'b'`


到這邊，我們大致能推出以下的優先級。
- 如果 `__get__` 與 `__set__` 都有，優先使用描述器（Descriptor）：
	- 當一個描述器同時實現了 `__get__` 和 `__set__` 方法時，Python 會認為它是一個資料描述器（Data Descriptor）。資料描述器的一個特點是它們對屬性的訪問有更高的優先級。
- 如果不是，則在自己的 `__dict__` 裡面找：
	- 如果描述器沒有作為資料描述器或者只實現了 `__get__` 方法的非資料描述器（Non-Data Descriptor）或者找不到描述器，Python 會繼續在物件的 `__dict__` 屬性字典中尋找是否存在該屬性。
- 若在 `__dict__` 也找不到，嘗試用描述器的 `__get__`：
	- 如果在物件的 `__dict__` 中找不到該屬性，Python 會檢查是否存在只實現了 `__get__` 方法的非資料描述器，如果存在，則回傳該描述器的 `__get__` 方法。
- 都沒有，直接回傳描述器：
	- 如果上述步驟都無法找到該屬性，最後會返回描述器物件本身，如果連描述器也不存在，則會拋出 AttributeError。


畫成圖:

![img](https://i.imgur.com/2tcc5aO.png)

## 範例一 - 兩者皆無

```python
class A:
    name = "meow"


o = A()
print(o.__dict__)
print(o.name)
```

> {{< collapse summary="output" >}}

```
{}
meow
```

{{</ collapse >}}



由於在運行 `o.name` 的時候，在本身的型別有找到 name 這個成員，但這個成員既沒有實現 `__get__` 也沒有實現 `__set__` 方法，該物件自身的 `__dict__` 也找不到 name。最後，就會直接回傳此成員。

## 範例二 - 被 `__dict__` 搶先 1

```python=
class Name:
    def __get__(self, obj, objtype):
        print("get trigger")
        return "meow"


class A:
    def __init__(self):
        self.name = Name()


o = A()
print(o.__dict__)
print(o.name)
```

> {{< collapse summary="output" >}}

```
{'name': <__main__.Name object at 0x000001F6BB0497F0>}
<__main__.Name object at 0x000001F6BB0497F0>
```

{{</ collapse >}}

在 line12 實例化 A 這個類別後，代表了 name 會被加到 o 的 `__dict__` 內。

在運行 `o.name` 後，由於發現在 o 的類型 A 中沒有發現 name 這個成員，接下來在 o 的 `__dict__` 中發現了 name，因此直接回傳。

## 範例三 - 被 `__dict__` 搶先 2

```python=
class Name:
    def __get__(self, obj, objtype):
        print("get trigger")
        return "meow"


class A:
    name = Name()


o = A()
print(o.name)
o.name = "wolf"
print(o.__dict__)
print(o.name)
```

> {{< collapse summary="output" >}}
```
get trigger
meow
{'name': 'wolf'}
wolf
```
{{</ collapse >}}

在 line11 實例化 A 這個類別產生了 o 物件後，line13 主動地將 `{name:"wolf"}` 放到 o 的 `__dict__` 中。

跟上一個範例類似，不同的是 name 有被定義在類別 A 內。
在運行 `o.name` 後，由於發現在 o 的類型 A 中有發現 name 這個成員，但只有實現 `__get__` 這個成員，沒有發現 `__set__` 因此現階段不會回傳，接下來在 o 的 `__dict__` 中發現了 name，因此直接回傳。

另外一個值得注意的是 `print("get trigger")` 並沒有被執行。

{{< info >}}
Note:
只有在同時定義了 `__get__` 與 `__set__` 的描述器才會在找自身的 `__dict__` 前被回傳，否則 `__dict__` 會先被查找使用。
{{< /info >}}

## 範例四 - 最高優先度的資料描述器

```python=
class Name:
    def __get__(self, obj, objtype):
        print("get trigger")
        return "meow"


class A:
    name = Name()


o = A()
print(o.name)
o.name = "wolf"
print(o.__dict__)
print(o.name)
Name.__set__ = lambda self, obj, value: None
print(o.name)
```

> {{< collapse summary="output" >}}

```
get trigger
meow
{'name': 'wolf'}
wolf
get trigger
meow
```
{{</ collapse >}}


這是一個反直覺的例子，我們在上個例子的 line13 加上了一行，也就是在 Name 這個類別中加上了一個任意的 `__set__` 函式，這樣在初次判斷描述器時，就會發現同時實現了 `__get__` 和 `__set__` 方法，因此就會直接回傳 `__get__` 的內容，看到結果是 `meow`

值得注意的是 `print("get trigger")` 有被執行。

## ref

> https://www.youtube.com/watch?v=Fp7aQO3QuS0&pp=ygUQcHl0aG9uIOaPj-i_sOWZqA%3D%3D
> https://openhome.cc/zh-tw/python/meta-programming/descriptor/
