---
title: python 处理 execl 文件
date: 2018-01-16 17:45:00
categories: python
tags: 
- 文件处理 
---

python 处理 excel 文件，需要用到第三方库: `xlrd` 和 `xlwt`。 运行 `pip install xlrd`, `pip install xlwt` 添加库支持。

<!--more-->

### `xlrd` 用来读取 excel, `xlwt` 用来创建并持久化一个 execl

```python

import xlrd
import xlwt

def process_execl():
    execl_data = xlrd.open_workbook('file_path')  # 打开一个 excel 文件
    sheet_list = execl_data.sheets()  # 获取 execl 有多少的 sheet, 返回一个 list

    for each_sheet in sheet_list:
        sheet_name = each_sheet.name
        print(sheet_name)
        rows = each_sheet.nrows  # 获取单个 sheet 的行数
        for i in range(rows):
            # execl 表格的第一行，一般是标题啥的，忽略
            if i == 0:
                continue
            row_value_list = each_sheet.row_values(i)  # 获取一行数据值，返回一个 list
            # 这样可以直接获取第 n 行，第 m 列的值
            # value = each_sheet.row_values(i)[col_num]

            for value in row_value_list:
                print(value)  # 获取每个值  # 写数据到 excel 文件
              
def write_excel():
    work_book = xlwt.Workbook()  # 创建一个 excel 表格
    sheet_1 = work_book.add_sheet('sheet_name')  # 添加一个 sheet
    sheet_1.write(0, 0, 'hello')  # 往 sheet_1 的第一行第一列写入一个数据
    sheet_1.write(0, 1, 'world')  # 往 sheet_1 的第一行第二列写入一个数据
    '''
    一次类推，填充 excel 的每个 sheet 的值 
    '''

    disk_file_path = "*.xls"  # 注意： 这里的后缀名为 .xls, 而非 .xlsx

    work_book.save(disk_file_path)  # 持久化到磁盘

```


以上代码就把可以轻松的解析和创建一个 execl 文件，是不是感觉屌。比 Java（辣鸡） 处理 execl 文件的程序简单易懂多了。 

接下来就可以转成自己想要的结构化的文件了，我比较喜欢转成 `json`, 当然， python 解析 `json`文件也是溜溜的。
