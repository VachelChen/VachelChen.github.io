---
layout: post
title: Python 应用 —— Excel交互
categories: Python
description: 有关python和Excel交互的一些实际操作。
keywords: Python, Excel
---

我在学生会有时面临复杂的数据要处理，我可没有耐心去人工核对，程序是很好实现的，尤其是使用Python，比如上次用了十几行的代码就从三百多个人中找出没有签到的人，涉及到了Python的集合减法。但其中我是使用把人名数据复制粘贴到IDLE里，作为input让程序跑，output也是在交互环境中简单的输出人名，这时我就想，如果能把Excel中的数据直接读入就好了，输出也成规范化的Excel就更好了，这一直是在我心理想去学习和实践的一个操作。

这次在中科院实习的一个“杂活”，给了我这个机会，一开始带我的研究生斌哥带我一起去老师那里听需求，听了半天我迷迷糊糊的因为毕竟生态环境所，讲的东西主要是化学反应上的，但后来梳理了一遍后发现并不难理解，关键就在于能把数据从Excel中提取并且输出。

![xiangmudata](/images/blog/xiangmudata.jpg)

话不多说，直接上程序：

```python

from xlutils.copy import copy	#用于拷贝格式
import xlrd						#用于读取Excel表格中的数据
import xlwt						#用于写入Excel表格数据
import sympy					#用于经行方程计算

# 格式设置
tem_excel = xlrd.open_workbook("model.xls", formatting_info=True)
tem_sheet = tem_excel.sheet_by_index(0)

new_excel = copy(tem_excel)
new_sheet = new_excel.get_sheet(0)

style = xlwt.XFStyle()
# 修改字体
font = xlwt.Font()
font.name = '等线'
font.height = 220
style.font = font

# 添加边框，THIN细线
borders = xlwt.Borders()
borders.top = xlwt.Borders.THIN
borders.bottom = xlwt.Borders.THIN
borders.left = xlwt.Borders.THIN
borders.right = xlwt.Borders.THIN
style.borders = borders

# 水平居中垂直居中
alignment = xlwt.Alignment()
alignment.horz = xlwt.Alignment.HORZ_CENTER
alignment.vert = xlwt.Alignment.VERT_CENTER
style.alignment = alignment

# 数据操作
data = xlrd.open_workbook("data.xlsx")
table = data.sheet_by_index(0)

# 厌氧池回流比计算
X = sympy.symbols("X")
getX = sympy.solve([(table.cell_value(4,1)+table.cell_value(4,12)*X)/(1+X) - table.cell_value(4,2)],[X])
print(getX[X])
X = round(getX[X],2)

# 厌氧池数据计算
yanO = []
for i in range(2, 14, 2):
    yanO.append(round((table.cell_value(i, 1) + table.cell_value(i, 12) * X) / (1 + X), 2))
print(yanO)


# 缺氧池回流比计算
Y = sympy.symbols("Y")
getY = sympy.solve([(table.cell_value(4,4)*(X+1)+table.cell_value(4,11)*Y)/(1+X+Y) - table.cell_value(4,6)],[Y])
print(getY[Y])
Y = round(getY[Y],2)

# 缺氧池数据计算
queO = []
for i in range(2, 14, 2):
    queO.append(round((table.cell_value(i, 11) *Y+ table.cell_value(i, 2) * (X+1)) / (1 + X + Y), 2))
print(queO)

# 填写理论值
for i in range(1,7):
    new_sheet.write(i,2,yanO[i-1],style)
for i in range(8,14):
    new_sheet.write(i,2,queO[i-8],style)

# 填写实际值
for i in range(1,7):
    new_sheet.write(i,3,table.cell_value(i*2,4),style)
for j in range(8,14):
    new_sheet.write(j,3,table.cell_value((j-7)*2,7),style)

# 填写回流比
new_sheet.write(1,4,X,style)
new_sheet.write(8,4,Y,style)

# 保存
new_excel.save('Value.xls')

```

简单来看就是分成了这么几个板块：

| 第三方库  |              |          		  			 |               			  |
| --------  | :--------    | :---------------- 			 | :------   				  |
| xlrd    	| 指定一个表格 | 指定一个sheet    			 | 读取每个cell的数据         |
| xlwt   	| 设置style	   | 写入每个cell数据 			 | 保存     			      |
| xlutils   | 拷贝表格     | 拷贝sheet         			 |                            |
| sympy     | 指定变量     | 初步整理方程并使用方法计算  |           				  |
