# Excel Manipulate

## Python XLRD (Excel Reader)

```python
import xlrd

workbook = xlrd.open_workbook('example.xls')
worksheet = workbook.sheet_by_index(0)

nrows = sheet.nrows

values = []

for row in range(nrows):
  values.append(worksheet.row_values(row))
```

## Python XLWR (Excel Writer)

```python
import xlwt

workbook = xlwt.Workbook(encoding='utf-8', cell_overwight_ok=True) # set default encoding
workbook.default_style.font.height = 20*11                         # for 11 pt text size

worksheet = workbook.add_sheet('Sheet Name') # create a sheet
font_style = xlwt.easyxf('font:height 280;')

worksheet.row(0).set_style(font_style)

worksheet.write_merge(0, 1, 0, 0, 'Excel example')
worksheet.write_merge(0, 0, 1, 2, 'Excel input')
worksheet.write_merge(0, 0, 3, 4, 'Excel output')

worksheet.write(1, 1, 'open')
worksheet.write(1, 2, 'read')
worksheet.write(1, 3, 'write')
worksheet.write(1, 4, 'store')

worksheet.save('example.xls')
```


---

*Reference from **[파이썬을 이용한 엑셀 파일 input & output - xlrd & xlwt](http://lorieliot.tistory.com/1 "Note")** by lorieliot*