---
title: "網路爬蟲環境設定（Python、pipenv、Vs Code）"
date: 2022-06-20
# weight: 1
# aliases: ["/first"]
tags: ["Crawler","develope environment"]
author: "Vincent55"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "「工欲善其事，必先利其器」廢話不多說，來開始準備網路爬蟲環境吧。"
canonicalURL: "https://blog.vincent55.tw/posts/crawlerenv/"
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
    image: "<image path/url>" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
# editPost:
#     URL: "https://github.com/<path_to_repo>/content"
#     Text: "Suggest Changes" # edit text
#     appendFilePath: true # to append file path to Edit link
---


## 前言
首先，這篇文其實是篇搬運文XD是從筆者之前參加的 [IThome 鐵人賽](https://ithelp.ithome.com.tw/m/users/20134430/ironman/4307) 中點閱最高的 [這篇文章](https://ithelp.ithome.com.tw/m/articles/10265084) 搬運過來的，當時參加的 IThome 鐵人賽是以網路爬蟲與反爬蟲技術作為主題，結果環境設定的文章與基礎語法是最受歡迎的🤣。

有興趣看網路爬蟲的其他文章可以 [點我](https://ithelp.ithome.com.tw/m/users/20134430/ironman/4307)


## Python3

python3 載點 : [https://www.python.org/downloads/](https://www.python.org/downloads/)

目前 Python 最新版為 3.9.7

![](https://i.imgur.com/iCy82ov.png)


▲下載時記得要將 Add Python 3.9 to PATH 開啟來才能在 cmd 等 command line 直接下 python 使用

![](https://i.imgur.com/NyDPvCU.png)


▲ Win + R 啟動執行，之後打入 `cmd` 打開命令提示字元

![](https://i.imgur.com/RCVFx8g.png)


▲ 打入 `python --version` 若有出現讀者下載 python 的版本則代表安裝成功

![](https://i.imgur.com/0ZrfIEn.png)


▲ 打入 `pip --version` 若有出現 pip 的版本則代表安裝成功（原版下載連結會同步把 pip 安裝好）

## pipenv 虛擬環境

有了 python 之後就需要來裝一些會使用到的套件，但在此之前，需要有個套件管理系統。讀者可能之前有聽說 pip ，pip 是一個用 python 寫的套件管理系統。以下是常用指令

```bash
pip install [package] #下載相對應的套件（package）
```

每下載一個套件，會預設裝到電腦的 python 安裝檔的 `/lib` 資料夾下，雖然下次開發其他專案時不用再下載，但有時會導致套件版本衝突的問題，就會需要頻繁維護 `requirements.txt`。

pipenv 是個虛擬開發環境的工具，會生成 `Pipfile` `Pipfile.lock` 並自動維護套件，當你要到其他環境開發時，只要 `pipenv install` 即可在虛擬環境中安裝 `Pipfile` 內的所有套件。

確保有安裝完成 `pip` 後，在 cmd 中打入 `pip install pipenv` 下載 pipenv

![](https://i.imgur.com/luFUm3E.png)


▲ 打入 pip install pipenv 下載 pipenv

![](https://i.imgur.com/kVv5jpD.png)


▲ 打入 pipenv —version 查看是否成功安裝且能執行

接下來到我們專案的資料夾，並在搜尋欄打入 `cmd` 即可"在這個路徑下開啟命令提示字元"

![](https://i.imgur.com/Zn5IL5c.png)


▲ 在欲開發專案的最外層目錄的搜尋欄打入 cmd 以在當前目錄開啟 cmd

接下來我們來裝 `requests` 這個套件測試。在上一步開啟的 cmd 中打入 `pipenv install requests` ，由於在當前目錄還沒虛擬環境，因此 pipenv 會先幫你建好一個，之後才會開始在這個虛擬環境安裝 `requests`這個套件。

![](https://i.imgur.com/PYpGWBM.png)


▲ 首次 pipenv install 在當前目錄建立虛擬環境並在虛擬環境內安裝套件

可以看到資料夾內自動生成了 `Pipfile` `Pipfile.lock` 兩個檔案，兩者都是拿來管理套件的檔案。

每個專案（套件虛擬環境）會有自己的 `Pipfile`，而`Pipfile.lock` 則是會有 `Pipfile` 裡面套件需要的其他套件（相依套件）以及各自的版本號，俗稱版本鎖。

※有興趣的讀者能用記事本打開 `Pipfile` `Pipfile.lock` 看一下這兩個檔案長怎樣歐

![](https://i.imgur.com/ytkxq6h.png)


▲ 每個專案執行 pipenv 產生虛擬環境後自動生成的套件管理文件 Pipfile Pipfile.lock

接下來打入 `pipenv shell` 進入虛擬環境來看看是否成功安裝 `requests` 這個套件。進入後打入 `python` 開啟 python 直譯器，在直譯器內打入 `import requests` 如果向下圖一樣不會報錯，就代表成功安裝套件。

![](https://i.imgur.com/Jq5LkbI.png)


▲ 檢測是否成功在虛擬環境內安裝套件

輸入 `quit()` 離開 python 直譯器，接下來輸入 `exit` 離開 pipenv 虛擬環境。 

![](https://i.imgur.com/Jkw3Vl2.png)


▲ 離開 python 直譯器與 pipenv 虛擬環境

接下來輸入 `pipenv --venv` 查看虛擬環境放置的路徑（此為套件、直譯器放置的位置，每個虛擬環境的路經皆不同），並將結果記下來 `C:\Users\50205\.virtualenvs\test-8TtvdmFW`，等等在 Vscode 中使用 pipenv 會使用到~

![](https://i.imgur.com/LyJNNPH.png)


▲ 輸入 pipenv —venv 查看虛擬環境放置路徑，沒意外的話會在C:\Users\%username%\.virtualenvs\test-8TtvdmFW

※有興趣的讀者能到 `pipenv --venv`給的路徑去看看套件的長相歐

了解 pipenv 基本操作後，接下來鐵人賽的內容將會頻繁使用 pipenv 來管理我們的套件，能在這裡找更多 pipenv 好用的功能

pipenv 指令大全 [:](https://hackmd.io/@sam-liaw/BJnLhni7U)　[https://medium.com/@hiimdoublej/pipenv指令大全-6e4415cc8a15](https://medium.com/@hiimdoublej/pipenv%E6%8C%87%E4%BB%A4%E5%A4%A7%E5%85%A8-6e4415cc8a15)

## Visual Studio Code

Vscode 是一款微軟開發且跨平台的免費原始碼編輯器，有許多非常好用的擴充功能，這邊會先請讀者下載，之後讓他能進入前面提到的 pipenv 虛擬環境，剩下的一些好用的擴充功能下面有附連結大家可以去安裝~

Vscode 載點 : [https://code.visualstudio.com/Download](https://code.visualstudio.com/Download)

Vscode 擴充套件推薦：[https://hackmd.io/@sam-liaw/BJnLhni7U](https://hackmd.io/@sam-liaw/BJnLhni7U)

![](https://i.imgur.com/Erp0y8H.png)


▲ 下載 Vscode 時記得將紅色的打勾

安裝完成後，先不要直接打開 Vscode。相同地，我們能在想要做專案的資料夾上方的搜尋欄打入 `code.cmd .` 就可以在當前目錄下開啟 Vscode。

![](https://i.imgur.com/SsBneZM.png)


▲ 在當前目錄下開啟 Vscode

之後先點擊左邊欄位 Extensions (1)，並在搜尋欄打入 `python` (2)點擊 `install` 下載 Vscode 的 Python extension(3)。

![▲ 下載 Vscode Python Extentions](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/fcf5724b-cadd-44d6-987d-e1e43c0f4d67/Untitled.png)

▲ 下載 Vscode Python Extentions

成功安裝 Vscode Python Extentions 後，我們要來設定在 Vscode 中自動進入 pipenv 的虛擬環境，在 Vscode 中按下 Ctrl+Shift+P 之後打入 `>Preferences: Open Settings (JSON)` ，之後在最後一行加入 `"python.venvPath": "{這邊放 pipenv --venv 所得的值(上面請大家記住的)}"`。

※大家要把每個 \ 之前再加上一個 \ 歐

![](https://i.imgur.com/fRgcToU.png)


▲ 在最後一行加入 pipenv 虛擬環境路徑

之後重啟 Vscode 之後點選左下角 Python 環境，就會發現可以選擇虛擬環境了~

![](https://i.imgur.com/MNlbhZI.png)


▲ 點擊圖中圈起來的虛擬環境，即可切換到 pipenv 的虛擬環境

---


## 補充資料

pipenv 指令大全 [:](https://hackmd.io/@sam-liaw/BJnLhni7U)　[https://medium.com/@hiimdoublej/pipenv指令大全-6e4415cc8a15](https://medium.com/@hiimdoublej/pipenv%E6%8C%87%E4%BB%A4%E5%A4%A7%E5%85%A8-6e4415cc8a15)

pipenv 簡潔小抄 [:](https://hackmd.io/@sam-liaw/BJnLhni7U)　[https://gist.github.com/bradtraversy/c70a93d6536ed63786c434707b898d55](https://gist.github.com/bradtraversy/c70a93d6536ed63786c434707b898d55)

Vscode 擴充套件推薦：[https://hackmd.io/@sam-liaw/BJnLhni7U](https://hackmd.io/@sam-liaw/BJnLhni7U)