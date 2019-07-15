---
layout: post
title: 软著申请——基于Django的行业学会科技奖评价管理系统
categories: Django
description: 系统正在筹划申请软著，分享一些申请书的填写
keywords: Django, applicant
---

## 申报材料准备



## 实现过程

**前置工作：**将 Word 文档空表格当作模板文档做好，与 Excel 数据源文件置于同一路径下。

```vb
Sub 分离()
    Application.ScreenUpdating = False
    
    p = ThisWorkbook.Path & "/"
    f = p & "空白模板.doc"
    
    Dim myWS As Worksheet
    Set myWS = ThisWorkbook.Sheets(1) '存有数据的表格
    
    For i = 3 To 54    '遍历数据行
        FileCopy f, p & "test/" & myWS.Cells(i, 2).Text & ".doc"    '复制空模板并以某列数据为名命名新产生的文档
        Set wd = CreateObject("word.application")
        Set d = wd.documents.Open(p & "test/" & myWS.Cells(i, 2).Text & ".doc") '打开新文档
        
        d.tables(1).Cell(1, 2) = myWS.Cells(i, 2).Text '###
        '复制表格每列内容到文档，有多少项就有多少条
        d.tables(1).Cell(5, 4) = myWS.Cells(i, 20).Text '###
        
        d.Close
        wd.Quit
        Set wd = Nothing
    Next
    
    Application.ScreenUpdating = True
End Sub
```
