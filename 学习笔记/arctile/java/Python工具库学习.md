## Python工具库学习

### pyechart

####  入门

```python
#  学习可视化报表pyecharts
# github地址 :  https://github.com/pyecharts/pyecharts
# 学习地址 : https://gallery.pyecharts.org/#/README
# 安装 : 到`venv/Scripts`路径下执行此命令 `pip install pyechartStart -U` 和 `pip install wheel`
# 生成的html在文件下,双击打开即可


from pyecharts.charts import Bar
from pyecharts import options as opts

# V1 版本开始支持链式调用
bar = (
    Bar()
    .add_xaxis(["衬衫", "毛衣", "领带", "裤子", "风衣", "高跟鞋", "袜子"])
    .add_yaxis("商家A", [114, 55, 27, 101, 125, 27, 105])
    .add_yaxis("商家B", [57, 134, 137, 129, 145, 60, 49])
    .set_global_opts(title_opts=opts.TitleOpts(title="某商场销售情况"))
)
bar.render()

# 不习惯链式调用的开发者依旧可以单独调用方法
# bar = Bar()
# bar.add_xaxis(["衬衫", "毛衣", "领带", "裤子", "风衣", "高跟鞋", "袜子"])
# bar.add_yaxis("商家A", [114, 55, 27, 101, 125, 27, 105])
# bar.add_yaxis("商家B", [57, 134, 137, 129, 145, 60, 49])
# bar.set_global_opts(title_opts=opts.TitleOpts(title="某商场销售情况"))
# bar.render()
```

```python
# 学习使用pyecharts来生成图片
# 安装 : 到`venv/Scripts`路径下执行此命令 `pip install snapshot-selenium`
# 遇到的问题参考 : https://blog.csdn.net/qq_35056459/article/details/90521942
#  第一个异常解决点 : 修改subprocess.py文件中__init__中的shell=False，如图找到_init_并修改False为True
#  第二个异常解决点 : 到http://npm.taobao.org/mirrors/chromedriver/下载对应的`chromedriver.exe`和程序放到一个工程目录下执行

from snapshot_selenium import snapshot as driver

from pyecharts import options as opts
from pyecharts.charts import Bar
from pyecharts.render import make_snapshot


def bar_chart() -> Bar:
    c = (
        Bar()
        .add_xaxis(["衬衫", "毛衣", "领带", "裤子", "风衣", "高跟鞋", "袜子"])
        .add_yaxis("商家A", [114, 55, 27, 101, 125, 27, 105])
        .add_yaxis("商家B", [57, 134, 137, 129, 145, 60, 49])
        .reversal_axis()
        .set_series_opts(label_opts=opts.LabelOpts(position="right"))
        .set_global_opts(title_opts=opts.TitleOpts(title="Bar-测试渲染图片"))
    )
    return c

# 需要安装 snapshot-selenium 或者 snapshot-phantomjs
make_snapshot(driver, bar_chart().render(), "bar.png")
```

#### 饼图

```python
from pyecharts import options as opts
from pyecharts.charts import Pie
from pyecharts.faker import Faker


# 数据也可以使用如下格式
# x_data = ["直接访问", "邮件营销", "联盟广告", "视频广告", "搜索引擎"]
# y_data = [335, 310, 274, 235, 400]

c = (
    Pie()
    .add("", [list(z) for z in zip(Faker.choose(), Faker.values())])
    .set_global_opts(title_opts=opts.TitleOpts(title="Pie-基本示例"))
    .set_series_opts(label_opts=opts.LabelOpts(formatter="{b}: {c}"))
    .render("pie_base.html")
)

```

```python
# 内嵌的饼图

import pyecharts.options as opts
from pyecharts.charts import Pie

inner_x_data = ["直达", "营销广告", "搜索引擎"]
inner_y_data = [335, 679, 1548]
inner_data_pair = [list(z) for z in zip(inner_x_data, inner_y_data)]

outer_x_data = ["直达", "营销广告", "搜索引擎", "邮件营销", "联盟广告", "视频广告", "百度", "谷歌", "必应", "其他"]
outer_y_data = [335, 310, 234, 135, 1048, 251, 147, 102]
outer_data_pair = [list(z) for z in zip(outer_x_data, outer_y_data)]

(
    Pie()
    .add(
        series_name="访问来源1111",     # 内部的名字无作用
        data_pair=inner_data_pair,
        radius=[0, "30%"],             # 定义内部饼图的半径范围,0-30%
        label_opts=opts.LabelOpts(position="inner"),
    )
    .add(
        series_name="访问来源",
        radius=["40%", "55%"],          # 定义外部饼图的范围为40-55%
        data_pair=outer_data_pair,
        label_opts=opts.LabelOpts(
            position="outside",
            formatter="{a|{a}}{abg|}\n{hr|}\n {b|{b}: }{c}  {per|{d}%}  ",
            background_color="#eee",
            border_color="#aaa",
            border_width=1,
            border_radius=4,
            rich={
                "a": {"color": "#999", "lineHeight": 22, "align": "center"},
                "abg": {
                    "backgroundColor": "#e3e3e3",
                    "width": "100%",
                    "align": "right",
                    "height": 22,
                    "borderRadius": [4, 4, 0, 0],
                },
                "hr": {
                    "borderColor": "#aaa",
                    "width": "100%",
                    "borderWidth": 0.5,
                    "height": 0,
                },
                "b": {"fontSize": 16, "lineHeight": 33},
                "per": {
                    "color": "#eee",
                    "backgroundColor": "#334455",
                    "padding": [2, 4],
                    "borderRadius": 2,
                },
            },
        ),
    )
    .set_global_opts(legend_opts=opts.LegendOpts(pos_left="left", orient="vertical"))   # 定义元素的图标位置垂直摆放
    .set_series_opts(
        tooltip_opts=opts.TooltipOpts(
            trigger="item", formatter="{a} <br/>{b}: {c} ({d}%)"
        )
    )
    .render("nested_pies.html")
)
```

```python
from pyecharts import options as opts
from pyecharts.charts import Pie
from pyecharts.faker import Faker

c = (
    Pie()
    .add(
        "",
        [list(z) for z in zip(Faker.choose(), Faker.values())],
        center=["35%", "50%"],
    )
    .set_global_opts(
        title_opts=opts.TitleOpts(title="Pie-调整位置"),
        legend_opts=opts.LegendOpts(pos_left="15%"),    # 调整图标位置
    )
    .set_series_opts(label_opts=opts.LabelOpts(formatter="{b}: {c}"))
    .render("pie_position.html")
)
```

#### 直方折线图

```python
from pyecharts import options as opts
from pyecharts.charts import Bar, Line
from pyecharts.faker import Faker

v1 = [2.0, 4.9, 7.0, 23.2, 25.6, 76.7, 135.6, 162.2, 32.6, 20.0, 6.4, 3.3]
v2 = [2.6, 5.9, 9.0, 26.4, 28.7, 70.7, 175.6, 182.2, 48.7, 18.8, 6.0, 2.3]
v3 = [2.0, 2.2, 3.3, 4.5, 6.3, 10.2, 20.3, 23.4, 23.0, 16.5, 12.0, 6.2]


bar = (
    Bar()
    .add_xaxis(Faker.months)
    .add_yaxis("蒸发量", v1)
    .add_yaxis("降水量", v2)
    .extend_axis(
        yaxis=opts.AxisOpts(
            axislabel_opts=opts.LabelOpts(formatter="{value} °C"), interval=5
        )
    )
    .set_series_opts(label_opts=opts.LabelOpts(is_show=False))  # 是否展示直方图上的数字
    .set_global_opts(
        title_opts=opts.TitleOpts(title="Overlap-bar+line"),
        yaxis_opts=opts.AxisOpts(axislabel_opts=opts.LabelOpts(formatter="{value} ml")),
    )
)

line = Line().add_xaxis(Faker.months).add_yaxis("平均温度", v3, yaxis_index=1)
bar.overlap(line)
bar.render("overlap_bar_line.html")
```

#### 指标图

```python

from pyecharts import options as opts
from pyecharts.charts import Gauge

c = (
    Gauge()
    .add("", [("完成率", 66.6)])
    .set_global_opts(title_opts=opts.TitleOpts(title="Gauge-基本示例"))
    .render("gauge_base.html")
)
```

### csv

```python
# 学习读csv数据


import csv
from collections import namedtuple

with open('stocks.csv') as f:
    # 读取csv的每一行内容
    reader = csv.reader(f)
    # 读取第一行header,此时reader的iterator定位到第二条记录,一般情况下返回的第一条记录为头部
    headers = next(reader)
    # 打印头部
    print(headers)
    # 使用header头部来关联下标,可达到直接通过row.Symbol来获取元素而不是通过下标`row[0]`来获取
    Row = namedtuple('Row', headers)
    # 重新遍历每一行,此时从第二行开始
    for r in reader:
        print(r)  # 输出每一行
        row = Row(*r)	# 将list的多个值转换为方法的入参
        print("当前行标记为" + row.Symbol)

print("==============使用字典的方式来访问元素==================")

# 使用字典的方式来访问csv的元素
with open('stocks.csv') as f:
    reader = csv.DictReader(f)
    for row in reader:
        print(row['Change'])


```

```python
# 学习写csv数据

import csv

headers = ['Symbol', 'Price', 'Date', 'Time', 'Change', 'Volume']
rows = [('AA', 39.48, '6/11/2007', '9:36am', -0.18, 181800),
        ('AIG', 71.38, '6/11/2007', '9:36am', -0.15, 195500),
        ('AXP', 62.58, '6/11/2007', '9:36am', -0.46, 935000),
        ]

# 注意这里的新一行设置为空,默认为空一行
with open('stocksWrite.csv', 'w', encoding='utf8', newline='') as f:
    # csv关联文件
    f_csv = csv.writer(f)
    # 写入头部
    f_csv.writerow(headers)
    # 写入数据
    f_csv.writerows(rows)

```

```python
# 学习往csv写入dict类型数据

import csv

headers = ['Symbol', 'Price', 'Date', 'Time', 'Change', 'Volume']
rows = [{'Symbol': 'AA', 'Price': 39.48, 'Date': '6/11/2007',
         'Time': '9:36am', 'Change': -0.18, 'Volume': 181800},
        {'Symbol': 'AIG', 'Price': 71.38, 'Date': '6/11/2007',
         'Time': '9:36am', 'Change': -0.15, 'Volume': 195500},
        {'Symbol': 'AXP', 'Price': 62.58, 'Date': '6/11/2007',
         'Time': '9:36am', 'Change': -0.46, 'Volume': 935000},
        ]

with open('stocksWriteDict.csv', 'w', encoding='utf8', newline='') as f:
    # csv文件关联头部
    f_csv = csv.DictWriter(f, headers)
    # 写头部
    f_csv.writeheader()
    # 写数据
    f_csv.writerows(rows)

```

### TinyDb

```python
# 学习TinyDb
# 官方文档 : https://github.com/msiemens/tinydb   https://tinydb.readthedocs.io/en/stable/
# 安装 : 在`venv\Scripts`文件夹下执行 `pip install tinydb`

from tinydb import TinyDB, Query, where

db = TinyDB('./db.json')

# 创建用户表
userTable = db.table('user')

# 删除全部
userTable.truncate()
print(userTable.all())

# 增加数据
userTable.insert({'age': 23, 'name': '刘备'})
userTable.insert({'age': 18, 'name': '张飞'})
userTable.insert({'age': 18, 'name': '诸葛亮'})
# 批量插入
userTable.insert_multiple([{'name': '关羽', 'age': 22}, {'name': '赵云', 'age': 17}, {'name': '黄忠', 'age': 50}])

# 查询有哪些表表
tableNums = db.tables()
print(tableNums)

# 查询user表的数据
print(userTable.all())

query = Query()
# 基于表格来查询
print(userTable.search(query.name == '刘备'))

# and 查询
print(userTable.search((query.name == '刘备') & (query.age <= 30)))

# and 查询
print(userTable.search((query.name == '刘备') | (query.age >= 22)))

# 更新操作
userTable.update({'age': 24}, where('name') == "刘备")
print(userTable.all())

# 无判断更新
userTable.update({'age': 0})
print(userTable.all())

# 删除操作
userTable.remove(where("name") == "黄忠")
print(userTable.all())


```

