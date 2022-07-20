---
title: "透過 Python 達成一般文書動作（word excel pdf printer）"
date: 2022-07-20
# weight: 1
# aliases: ["/first"]
tags: ["python","paperwork","生活智慧王"]
author: "Vincent55"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "紀錄一下我是怎麼透過程式來完成一般文書作業的"
canonicalURL: "https://blog.vincent55.tw/posts/paperworkwithprogram/"
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
因為有案子的需求要使用自動化的文書動作，因此這篇 blog 來記錄一下我用了哪些工具來達到這些功能，內容蠻雜的。

## Pdf

### 判斷 pdf 是否可被開啟（是否沒損毀）
https://pypi.org/project/PyPDF2/

```python
from PyPDF2 import PdfWriter, PdfReader
    def check_pdf_vaild(self):
        try:
            PdfReader(self.filename)
            return True
        except:
            return False
```

### pdf 轉 png
https://pypi.org/project/pdf2image/
此套件為 poppler 的封裝，請安裝 poppler。
```python
from pdf2image import convert_from_path
images = convert_from_path(f'{file_path}.pdf')
images[0].save(f"{file_path}.png")
```

此 images 為 pillow 物件，因此如果同時要剪裁，可以有以下操作
```python
from pdf2image import convert_from_path
images = convert_from_path(f'{file_path}.pdf')
images[0].crop((300, 50, images[0].size[0]/2, images[0].size[1]/2.55)).save(f"{file_path}.png")
```

### pdf 剪裁
https://pypi.org/project/PyPDF2/

```python
from PyPDF2 import PdfWriter, PdfReader
reader = PdfReader(f'{file_path}.pdf')
writer = PdfWriter()
page = reader.pages[0]
ur = page.cropbox.upper_right
page.cropbox.lower_left = (110, float(ur[1])/1.6)
page.cropbox.upper_right = (float(ur[0])/2, float(ur[1])/1.015)
writer.add_page(page)
with open(f'{file_path}.pdf','wb') as fp:
    writer.write(fp)
```

### pdf 分拆為一頁一個檔案

https://pypi.org/project/PyPDF2/

```python
from PyPDF2 import PdfWriter, PdfReader
inputpdf = PdfFileReader(open(self.pdf_file_path, "rb"))

for idx, split_pdf_file_path in enumerate(self.split_pdf_file_paths):
    output = PdfFileWriter()
    output.addPage(inputpdf.getPage(idx))
    with open(split_pdf_file_path, "wb") as outputStream:
        output.write(outputStream)
```


### html（url） 轉 pdf
https://pdfkit.org/

此工具可支援用 POST 且自訂 header
```python
import pdfkit
options = {'custom-header': [
        ('Host','external2.shopee.tw'),
        ('Content-Type','application/x-www-form-urlencoded')
    ],'post':[('ApiKey', re.search(r'name="ApiKey" value="(.*)"', fileserver_text).group(1)),
            ('Data', html.unescape(re.search(r'name="Data" value="(.*)"', fileserver_text).group(1))),
            ('TimeStamp', re.search(r'name="TimeStamp" value="(.*)"', fileserver_text).group(1)),
            ('Signature', re.search(r'name="Signature" value="(.*)"', fileserver_text).group(1))]}

pdfkit.from_url(f"{shopp_url}/OrderPrint.aspx", f"{self.filename}.pdf",options=options)
```

## Excel

### Excel 操作
https://openpyxl.readthedocs.io/en/stable/
```python
from openpyxl import load_workbook

template_excel = load_workbook(excel_database_file)
template_excel_actsheet = template_excel.active
rows = tuple(self.template_excel_actsheet.iter_rows())
email_col_idx = [r.value for r in rows[0]].index('email_column')
for row in reversed(rows): # for delete row
    if row[0].row == 1:
        continue
        
    if not self.email_check(row[email_col_idx].value):
        template_excel_actsheet.delete_rows(row[0].row)
template_excel.save(gen_excel_file)
template_excel.close()
```

## Word

### word 生成 html
https://pypi.org/project/mammoth/
```python
import mammoth
custom_styles = """
            h1 => p
            strong => p
            """
with open(self.word_file_path, "rb") as docx_file:
    result = mammoth.convert_to_html(docx_file, style_map = custom_styles, include_default_style_map=False)
    # 為了要讓表格 css 出的來
    doc_texts = result.value.replace("<table>", '<table style="border: 1px solid black;">').replace("<th>", '<th style="border: 1px solid black;>').replace("<td>", '<td  style="border: 1px solid black;">')
```

### word mailmerge
https://pypi.org/project/pywin32/
請先使用 word 內建 mailmerge 功能進行手動 mailmerge 設定
```python
import win32com.client
class mailmerger():
    def __init__(self, template_docx_path, xlsx_path, docx_file, pdf_file)
        self.template_docx_path = template_docx_path
        self.xlsx_path = xlsx_path
        self.docx_file = docx_file
        self.pdf_file = pdf_file
    
    def setting_template_word(self):
        self.docx = win32com.client.Dispatch("Word.Application")
        self.docx.Visible = True
        self.docx.DisplayAlerts = 0

    def open_document(self):
        self.document = self.docx.Documents.Open(template_docx_path)

    def run(self):
        self.document.MailMerge.OpenDataSource(
                    xlsx_path,
            SQLStatement="SELECT * FROM `'client list$'`")

        self.document.MailMerge.Execute()
        self.document.MailMerge.Application.ActiveDocument.TrackRevisions = False
        self.document.MailMerge.Application.ActiveDocument.Revisions.AcceptAll()
        document_pdf = self.docx.Documents(1)
        document_pdf.SaveAs(docx_file)
        document_pdf.SaveAs(pdf_file, 17)
        document_pdf.Close()
```

## Email

### 確認 email 格式（regex）
```python
import re
def email_check(self, email):
    regex = r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b'
    if email == None:
        return False
    if re.fullmatch(regex, email):
        return True
    else:
        return False
```

## 印表機 on Windows

### windows 透過 CLI 列印 pdf 到預設印表機
透過 SumatraPDF.exe
https://www.sumatrapdfreader.org/download-free-pdf-viewer
下載後，將 SumatraPDF.exe 單獨拿出來使用（經測試後，不需要其他 utils）

```python
import win32api
win32api.ShellExecute(0, 'open', self.SumatraPDF_PATH, f' -print-to-default -print-settings "paper={paper_size},fit"  "{self.filename}"', '.', 0)
```

### windows 取得預設印表機是否 offline
```python
import win32print

def chk_offline(self):
    currentprinter = win32print.GetDefaultPrinter()
    handle = win32print.OpenPrinter(currentprinter)
    attributes = win32print.GetPrinter(handle)[13]
    return (attributes & 0x00000400) >> 10
```


## 一般

### 重製資料夾
```python
import os
import shutil

gen_excel_folder = os.path.abspath(self.gen_excel_folder)
if os.path.exists(gen_excel_folder):
    shutil.rmtree(gen_excel_folder)
os.mkdir(gen_excel_folder)
```