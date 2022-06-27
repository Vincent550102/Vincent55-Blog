---
title: "如何在 Windows 設定開機自動啟動（含 Powershell 腳本、不須設定服務）"
date: 2022-06-27
# weight: 1
# aliases: ["/first"]
tags: ["Windows"]
author: "Vincent55"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "如何像 Linux 那樣設定開機自動啟動呢"
canonicalURL: "https://blog.vincent55.tw/posts/windowsstartup/"
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
Linux 的開機自動啟動是非常常見且易用的，只需要將服務設定 systemd 之後設定 enable 即可。相對比較少人在用的是 Windows 的開機自動啟動，但其實 Windows 的開機自動啟動也是屬於容易設定的。

---
## 前言
Linux 的開機自動啟動是非常常見且易用的，只需要將服務設定 systemd 之後設定 enable 即可。相對比較少人在用的是 Windows 的開機自動啟動，但其實 Windows 的開機自動啟動也是屬於容易設定的。

---

## 正文

### 開啟 Startup 資料夾
按下 win+R 並輸入 `shell:startup` 則會開啟 Startup 資料夾。
![](https://i.imgur.com/533iEgy.png)
![](https://i.imgur.com/QUkh2pf.png)

### 設定開機自動啟動
將想要開機自動啟動的應用程式的 **捷徑** 放入 Startup 資料夾中。

:::info
注意，請確保你放的是捷徑
:::
![](https://i.imgur.com/uIXwEwY.png)

### 檢查是否成功設定
開啟工作管理員後，切到開機分頁，如果剛才放入的應用程式捷徑出現在這，並且狀態為已啟用，代表已經設定完成。
![](https://i.imgur.com/PBEyHxR.png)

---

## 同場加映（如何設定開機自動啟動 Powershell 腳本）
### 安全政策
微軟為了控制 Powershell 載入設定檔和執行腳本時所使用的條件，限制了 powershell 腳本，也就是 `*.ps1` 的執行場景，詳細可看[官方文件](https:/go.microsoft.com/fwlink/?LinkID=135170)。

例如說我有一隻含以下內容的 powershell 腳本。

owo.ps1
```powershell
echo "owo";
```
之後我在 cmd 嘗試執行它，會發現有以下錯誤。
![](https://i.imgur.com/3R7Us9s.png)

並且，各位可以嘗試直接將該腳本的捷徑放至上述的 Startup 資料夾，會發現開機後並不會成功執行，而是會將該腳本以 notepad 開啟（這是因為 `*.ps1` 預設為 "notepad 開啟"）。

這時，或許你會想到用一個 `*.bat` 腳本來跑 `*.ps1` 但這就會遇到上述安全政策的問題。

### 設定 Powershell 執行 policies

使用以下腳本即可成功在 `*.bat` 中執行 `*.ps1`。
```
Powershell.exe -executionpolicy remotesigned -File <.ps1>
```

更多 policy 的內容可以參考[官方文件](https://docs.microsoft.com/zh-tw/powershell/module/microsoft.powershell.core/about/about_execution_policies?view=powershell-7.2)

最後，只需將該 `*.bat` 放置於 Startup 資料夾即可完成開機自動啟動 Powershell 腳本指令。

