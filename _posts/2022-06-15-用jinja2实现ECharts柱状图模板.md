---
title: 用 jinja2 实现 ECharts 柱状图模板
date: 2022-06-15
categories: 
- Python
tags:
- jinja2
- echarts
- pyecharts
---


# 背景
在上一篇文章中 [从源码解析 pyecharts 如何生成一个柱状图](https://cn-lanbao.github.io/python/2022/06/14/%E4%BB%8E%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90pyecharts%E5%A6%82%E4%BD%95%E7%94%9F%E6%88%90%E4%B8%80%E4%B8%AA%E6%9F%B1%E7%8A%B6%E5%9B%BE/) 发现 pyecharts 和 flask 一样用到了 Jinja2，本文就简单学习 Jinja2，目的是为了理解 pyecharts 是如何构建 html 的。惯例，先附上 [中文文档](http://docs.jinkan.org/docs/jinja2/)（感谢 yinian1992@gmail.com 的翻译） ，但是请注意：**官方文档是为 Python2 编撰的，但是本文基于 Python3**  

整体的思路是：获取生成 html 的参数值 -> 填充到 html 模板，获取 html 内容 -> 写入 html 文件  
首先需要了解基础用法和宏，注意，以下示例仅介绍了冰山一角的冰山一角，仅为能理解 pyecharts 做的基础学习，请务必查看官方文档


# 简单用法
目录结构
```
Demo
|
|--templates
|   |
|   |--test.html  # 模板
|    
|--main.py  # 测试代码
```

`templates/test.html`
```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
<p>{{ name }}</p>
</body>
</html>
```

`main.py`，加载模板目录，并通过 `get_template` 获取指定模板，`render` 将数据传递给模板，获取到的 html 字符串写入文件，这样便通过模板生成了一个 html 文件 
```
from pathlib import Path
from jinja2 import Environment, FileSystemLoader


env = Environment(loader=FileSystemLoader(Path(__file__).parent / "templates"))
template = env.get_template('test.html')
html = template.render(name="CN-LanBao")
with open("test.html", "w+", encoding="utf-8") as html_file:
    html_file.write(html)

```

`test.html`
```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
<p>CN-LanBao</p>
</body>
</html>
```


# 宏
> 宏类似常规编程语言中的函数。它们用于把常用行为作为可重用的函数，取代手动重复的工作  

在 templates 目录下创建 macro 文件，`templates/macro`
```
{% raw %}
{% macro p(name) -%}
    <p>{{ name }}</p>
{%- endmacro %}
{% endraw %}
```

修改一下 `templates/test.html` 的代码
```
{% raw %}
{% import 'macro' as macro %}
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
{{ macro.p(name) }}
</body>
</html>
{% endraw %}
```

再次执行 `main.py` 生成 `test.html`
```

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
<p>CN-LanBao</p>
</body>
</html>
```
用起来和 Python 的函数一样，还是非常好理解的


# 实际操作
在看 pyecharts 之前，先尝试自己写一个简单的柱状图模板  
先去 ECharts 官网的 [快速入门手册](https://echarts.apache.org/handbook/zh/get-started/) 中复制一个柱状图的完整 html 代码，并稍作修改存放在我们的 `templates/test.html` 里
```
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8" />
    <title>ECharts</title>
    <!-- 换成在线依赖，省去下载至本地的步骤了，但是必须联网才能查看 -->
    <script type="text/javascript" src="https://assets.pyecharts.org/assets/echarts.min.js"></script>
  </head>
  <body>
    <!-- 为 ECharts 准备一个定义了宽高的 DOM -->
    <div id="main" style="width: 600px;height:400px;"></div>
    <script type="text/javascript">
      // 基于准备好的dom，初始化echarts实例
      var myChart = echarts.init(document.getElementById('main'));

      // 指定图表的配置项和数据
      var option = {
        title: {
          text: 'ECharts 入门示例'
        },
        tooltip: {},
        legend: {
          data: ['销量']
        },
        xAxis: {
          data: ['衬衫', '羊毛衫', '雪纺衫', '裤子', '高跟鞋', '袜子']
        },
        yAxis: {},
        series: [
          {
            name: '销量',
            type: 'bar',
            data: [5, 20, 36, 10, 10, 20]
          }
        ]
      };

      // 使用刚指定的配置项和数据显示图表。
      myChart.setOption(option);
    </script>
  </body>
</html>

```

为了可拓展性，我们需要将存放 ECharts 图的这块逻辑用宏代替，即写入到 `templates/macro` 中。我们将会用一个类来代替上文示例中的字典承载数据
```
{% raw %}
{% macro render_bar(bar) -%}
    <div id="main" style="width: {{ bar.opts.width }}px;height: {{ bar.opts.height }}px;"></div>
    <script type="text/javascript">
        var myChart = echarts.init(document.getElementById('main'));

        var option = {
            title: {
                text: '{{ bar.opts.title }}'
            },
                tooltip: {},
            legend: {
                data: {{ bar.opts.legend }}
            },
            xAxis: {
                data: {{ bar.opts.x_axis }}
            },
            yAxis: {},
            series: {{ bar.opts.series }}
        };

        myChart.setOption(option);
    </script>
{%- endmacro %}
{% endraw %}
```
写好 `templates/macro` 后，`templates/test.html` 只需要调用 macro 了
```
{% raw %}
{% import 'macro' as macro %}
<!DOCTYPE html>
<html>
    <head>
        <meta charset="utf-8" />
        <title>ECharts</title>
        <!-- 换成在线依赖，省去下载至本地的步骤了，但是必须联网才能查看 -->
        <script type="text/javascript" src="https://assets.pyecharts.org/assets/echarts.min.js"></script>
    </head>
    <body>
    {{ macro.render_bar(bar) }}
    </body>
</html>
{% endraw %}
```

最后就是我们的 `main.py`，新建一个 `Bar` 类用于承载数据，之后将实例对象作为 `render` 的参数
```
from pathlib import Path
from jinja2 import Environment, FileSystemLoader


class Bar(object):
    __slots__ = ("opts",)

    def __init__(self):
        self.opts = {
            "width": 600,
            "height": 400,
            "title": "CN-LanBao 测试柱状图",
            "legend": [],
            "series": [],
            "x_axis": [],
        }


env = Environment(loader=FileSystemLoader(Path(__file__).parent / "templates"))
template = env.get_template('test.html')
bar = Bar()
bar.opts["legend"] = ["销量"]
bar.opts["x_axis"] = ["衬衫", "羊毛衫", "雪纺衫", "裤子", "高跟鞋", "袜子"]
bar.opts["series"] = [{"name": "销量", "type": "bar", "data": [5, 20, 36, 10, 10, 20]}]


html = template.render(bar=bar)
with open("test.html", "w+", encoding="utf-8") as html_file:
    html_file.write(html)
```

生成出来的 `test.html` 和浏览器查看的效果
```

<!DOCTYPE html>
<html>
    <head>
        <meta charset="utf-8" />
        <title>ECharts</title>
        <!-- 换成在线依赖，省去下载至本地的步骤了，但是必须联网才能查看 -->
        <script type="text/javascript" src="https://assets.pyecharts.org/assets/echarts.min.js"></script>
    </head>
    <body>
    <div id="main" style="width: 600px;height: 400px;"></div>
    <script type="text/javascript">
        var myChart = echarts.init(document.getElementById('main'));

        var option = {
            title: {
                text: 'CN-LanBao 测试柱状图'
            },
                tooltip: {},
            legend: {
                data: ['销量']
            },
            xAxis: {
                data: ['衬衫', '羊毛衫', '雪纺衫', '裤子', '高跟鞋', '袜子']
            },
            yAxis: {},
            series: [{'name': '销量', 'type': 'bar', 'data': [5, 20, 36, 10, 10, 20]}]
        };

        myChart.setOption(option);
    </script>
    </body>
</html>
```
![avatar][01]


# pyecharts
终于到这一步了，html 模板大同小异，主要看一下它的 macro
```
{% raw %}
{%- macro render_chart_content(c) -%}
    <div id="{{ c.chart_id }}" class="chart-container" style="width:{{ c.width }}; height:{{ c.height }};"></div>
    <script>
        var chart_{{ c.chart_id }} = echarts.init(
            document.getElementById('{{ c.chart_id }}'), '{{ c.theme }}', {renderer: '{{ c.renderer }}'});
        {% for js in c.js_functions.items %}
            {{ js }}
        {% endfor %}
        var option_{{ c.chart_id }} = {{ c.json_contents }};
        chart_{{ c.chart_id }}.setOption(option_{{ c.chart_id }});
        {% if c._is_geo_chart %}
            var bmap = chart_{{ c.chart_id }}.getModel().getComponent('bmap').getBMap();
            {% if c.bmap_js_functions %}
                {% for fn in c.bmap_js_functions.items %}
                    {{ fn }}
                {% endfor %}
            {% endif %}
        {% endif %}
        {% if c.width.endswith('%') %}
            window.addEventListener('resize', function(){
                chart_{{ c.chart_id }}.resize();
            })
        {% endif %}
    </script>
{%- endmacro %}
{% endraw %}
```
有了前面的基础，这段代码看起来还是不那么困难的  
显而易见，为了支持不同类型的 ECharts 图的结构体，引入 `if` 和 `for`。并且在类的属性中，就已经完成了大部分的数据格式转化，macro 仅需要完成数据填充  
感兴趣的可以具体查看一下 pyecharts 是如何去设计类的




[01]:data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAk8AAAF6CAYAAAAAmLztAAAAAXNSR0IArs4c6QAAAARnQU1BAACxjwv8YQUAAAAJcEhZcwAADsMAAA7DAcdvqGQAADPjSURBVHhe7d37j6VZXe/x8y+ZiSEe4w+QI1HT5GjUKETpaDQalcbLGC+txhuOXCTaYoIOo1Gi4A1tryhqRkQOkm7xwkRBbHRGYdTRRh1tFHCffDb1LVav/dlP1a5vPTNrr+/7lXzSVXvvWrX3em6ffvZTVf9rAwAAgHOjPAEAAByA8gQAAHAAyhMAAMABKE8AAAAHoDwBAAAcgPIEAABwAMoTAADAAShPAAAAB6A8YVgf+9jHNv/zP/9z8hkAAGOgPBXw13/915tf/MVf3PzXf/3X5qMf/ejmX//1Xzd3797dyb17906+YrN9zLd8y7dso4/ln//5nze//Mu/vPmP//iP7ef73LhxY/PAAw9srl+/vv3eb3rTm7bft3Xz5s3tY/TY3tNPP715/etfv/mcz/mczdvf/vaTW/fT89b30ngatxXfR/f/53/+5+axxx7b/Nmf/dnJvR8vaPvm4zzR12oMPLM+9pGPnHwEAM+8SylP//7v/775jd/4jc03fuM3bl784hdvc+3atc1P/uRPbp566qntY971rndtXvva127z8z//8/cdqIMOyj/90z+9fczv/d7vndx6tn/8x3/c/PiP//j2637lV35l89///d8n9zwz2u/f5uGHH978wR/8wWn5eDY8/vjjmy/4gi/YFojv+I7v2Ny+fXtz5cqV7ed9VDTe9773bW7durV54oknNlevXt1GJeHJJ5/cfPEXf/H2cW984xsXzwipLOlx+lp9zXOe85zNj/3Yj91XoJbKU1uGvvIrv3Jb2pactzxpWXzap33atpSp1Ilem56nHnORxPxg2fte+fLN/3v+8zZv/eRPurRoPI275M6dO5uHHnpoZ3+j9aJfV6Rdl1xY3gAkVZ70P+7f/M3f3Dz/+c+3OxpFB2uJg5iig+mv/dqvbW9vtQcyd1DdRzvIKATa8blitqb2+7t8yqd8yvb1PhtvQWkZae71HPRcvv7rv37zGZ/xGdtlFkX3cz/3c7f36XGad70WLTctizhY6Lnrfi275z73ufedvenFstayePe7370tKxrn7/7u704esVye5E//9E+3RUff71d/9VdP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