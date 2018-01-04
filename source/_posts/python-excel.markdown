---
title: python 处理 execl 文件
date: 2017-12-25 15:06:00
categories: python
-tags: 文件处理 
---

python 处理 excel 文件，需要用到一个第三方库: `xlrd`。 运行 `pip install xlrd` 添加库支持。

<!--more-->

废话少说，直接上代码.

```python
import xlrd

def process_execl():
    execl_data = xlrd.open_workbook('file_path') # 打开一个 excel 文件
    sheet_list = excel_data.sheets() # 获取 execl 有多少的 sheet, 返回一个 list
    for each_sheet in sheet_list:
        sheet_name = each_sheet.name
        rows = each_sheet.nrows  # 获取单个 sheet 的行数
        for i in range(rows):
            if == 0:  # execl 表格的第一行，一般是标题啥的，忽略
                continue
            row_value_list = each_sheet.row_values(i)  # 获取一行数据值，返回一个 list
            # 这样可以直接获取第 n 行，第 m 列的值
            # value = each_sheet.row_values(i)[col_num]
            
            for value in row_value_list:
                print(value) # 获取每个值
``` 

以上代码就把可以轻松的解析一个 execl 文件，是不是感觉屌。比 Java（辣鸡） 处理 execl 文件的程序简单易懂多了。 

接下来就可以转成自己想要的结构化的文件了，我比较喜欢转成 `json`, 当然， python 解析 `json`文件也是溜溜的。