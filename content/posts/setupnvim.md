---
title: "在 Debian 安裝 Neovim 並設定 AstroNvim 自訂設定檔"
date: 2023-07-08
# weight: 1
# aliases: ["/first"]
tags: ["neovim", "setup"]
author: "Vincent55"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "紀錄一下在 Debian 作業系統上安裝 Neovim 並設定 AstroNvim 自訂設定檔"
canonicalURL: "https://blog.vincent55.tw/posts/setupnvim/"
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

## 安裝 Neovim

請先確保你的 Terminal 有 [Nerd Fonts](https://www.nerdfonts.com/font-downloads)

```bash
sudo apt-get update
sudo apt-get -y install fuse git python3-venv unzip ripgrep fzf

# install nvm https://github.com/nvm-sh/nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.5/install.sh | bash

# install npm
nvm install node
npm i -g eslint

# install nvim
wget https://github.com/neovim/neovim/releases/download/stable/nvim.appimage
chmod u+x nvim.appimage && sudo mv ./nvim.appimage /usr/bin/nvim
```

## 設定 AstroNvim

### 安裝 AstroNvim

```bash
git clone --depth 1 https://github.com/AstroNvim/AstroNvim ~/.config/nvim && nvim
```

### 匯入自訂設定檔

```bash
git clone https://github.com/Vincent550102/nvim-userconf ~/.config/nvim/lua/user && nvim
```
