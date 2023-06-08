---
title: 从源码解析 pyecharts 如何生成一个柱状图
date: 2022-06-14
updated: 2022-06-15
categories: 
- Python
tags:
- echarts
- pyecharts
---


# 背景
从实习 Java web 后端开发开始就在用 [ECharts](https://echarts.apache.org/zh/index.html) 做数据可视化了，到现在做效能相关工作在开源平台 Grafana 上使用。为同事实现的性能测试数据生成绘制图表也是用了 ECharts，但是过程较为繁琐，需要往自己准备的 html 模板中写入对应的数据  
最近发现有个国人开发的库 [pyecharts](https://github.com/pyecharts/pyecharts) ，将 Python 和 ECharts 相结合，[中文文档](https://pyecharts.org/#/zh-cn/)   

将其作为自己第一个学习的开源项目，以生成柱状图为例，对其源码进行解析

解析流程通过 Pycharm 的 debug 进行，目标为官方的第一条示例代码  
```
from pyecharts.charts import Bar

bar = Bar()
bar.add_xaxis(["衬衫", "羊毛衫", "雪纺衫", "裤子", "高跟鞋", "袜子"])
bar.add_yaxis("商家A", [5, 20, 36, 10, 75, 90])
# render 会生成本地 HTML 文件，默认会在当前目录生成 render.html 文件
# 也可以传入路径参数，如 bar.render("mycharts.html")
bar.render()
```


# 初始化
首先 `Bar` 类的相对路径是 `pyecharts/charts/basic_charts/bar.py`，为什么可以不通过 `from pyecharts.charts.basic_charts.bar import Bar` 去 import 呢？因为 `pyecharts/charts` 目录下的 `__init__.py` 文件中实现了 `from ..charts.basic_charts.bar import Bar`  


接下来通过 `print(Bar.__mro__)` 来看一下 `Bar` 的继承关系
```
(<class 'pyecharts.charts.basic_charts.bar.Bar'>, <class 'pyecharts.charts.chart.RectChart'>, 
<class 'pyecharts.charts.chart.Chart'>, <class 'pyecharts.charts.base.Base'>, 
<class 'pyecharts.charts.mixins.ChartMixin'>, <class 'object'>)
```

目前是创建类实例的阶段，暂且先只看每个类实例化相关内容  

从最上级（除了 `object`）的 `ChartMixin` 类开始看起，只定义了两个方法 `add_js_funcs` 和 `load_javascript` 从命名来看和前端有关系


然后是 `Base`，官方的注释是
> `Base` is the root class for all graphical class, it provides part of the initialization parameters and common methods  
> `Base` 是所有图形类的根类，它提供部分初始化参数和常用方法

```
class Base(ChartMixin):
    def __init__(self, init_opts: Union[InitOpts, dict] = InitOpts()):
        _opts = init_opts
        if isinstance(init_opts, InitOpts):
            _opts = init_opts.opts

        self.width = _opts.get("width", "900px")
        self.height = _opts.get("height", "500px")
        self.renderer = _opts.get("renderer", RenderType.CANVAS)
        self.page_title = _opts.get("page_title", CurrentConfig.PAGE_TITLE)
        self.theme = _opts.get("theme", ThemeType.WHITE)
        self.chart_id = _opts.get("chart_id") or uuid.uuid4().hex

        self.options: dict = {}
        self.js_host: str = _opts.get("js_host") or CurrentConfig.ONLINE_HOST
        self.js_functions: utils.OrderedSet = utils.OrderedSet()
        self.js_dependencies: utils.OrderedSet = utils.OrderedSet("echarts")
        self.options.update(backgroundColor=_opts.get("bg_color"))
        self.options.update(_opts.get("animationOpts", AnimationOpts()).opts)
        self._is_geo_chart: bool = False
```

可以看到设置了一个默认入参 `InitOpts()`，`InitOpts` 继承于 `BasicOpts`。而 `BasicOpts` 比较特殊，实现了 `__slots__`，说明这是一个只被当作数据结构而频繁使用的类，这样的类会大量减少内存的使用（可以参考我的另外一篇文章 [类与对象（上）](https://lanbaoshen.github.io/python/2022/05/13/%E7%B1%BB%E4%B8%8E%E5%AF%B9%E8%B1%A1-%E4%B8%8A/) 中的**当创建大量实例时如何节省内存**）

```
class BasicOpts:
    __slots__ = ("opts",)
```

`InitOpts` 在 `__init__` 中完成了属性 `opts` 的赋值，初始化了一堆参数

```
class InitOpts(BasicOpts):
    def __init__(
        self,
        width: str = "900px",
        height: str = "500px",
        chart_id: Optional[str] = None,
        renderer: str = RenderType.CANVAS,
        page_title: str = CurrentConfig.PAGE_TITLE,
        theme: str = ThemeType.WHITE,
        bg_color: Union[str, dict] = None,
        js_host: str = "",
        animation_opts: Union[AnimationOpts, dict] = AnimationOpts(),
    ):
        self.opts: dict = {
            "width": width,
            "height": height,
            "chart_id": chart_id,
            "renderer": renderer,
            "page_title": page_title,
            "theme": theme,
            "bg_color": bg_color,
            "js_host": js_host,
            "animationOpts": animation_opts,
        }
```
再回到 `Base` 类中，通过 `isinstance()` 判断 `_opts` 是否为 `InitOpts` 类的实例，之后从其属性 `opts` 字典中获取值初始化 `Base` 的属性


接下来是 `Chart`
```
class Chart(Base):
    def __init__(self, init_opts: types.Init = opts.InitOpts()):
        if isinstance(init_opts, dict):
            temp_opts = opts.InitOpts()
            temp_opts.update(**init_opts)
            init_opts = temp_opts
        super().__init__(init_opts=init_opts)
        self.colors = (
            "#c23531 #2f4554 #61a0a8 #d48265 #749f83 #ca8622 #bda29a #6e7074 "
            "#546570 #c4ccd3 #f05b72 #ef5b9c #f47920 #905a3d #fab27b #2a5caa "
            "#444693 #726930 #b2d235 #6d8346 #ac6767 #1d953f #6950a1 #918597"
        ).split()
        if init_opts.opts.get("theme") == ThemeType.WHITE:
            self.options.update(color=self.colors)
        self.options.update(
            series=[],
            legend=[{"data": [], "selected": dict()}],
            tooltip=opts.TooltipOpts().opts,
        )
        self._chart_type: Optional[str] = None
```
同样的，设置 `InitOpts()` 为默认参数供父类 `Base` 作部分参数的初始化。如果接收到的 `init_opts` 是一个字典，则实例化一个 `InitOpts` 对象，并用 `init_opts` 的值更新属性 `opts` 字典内的值  
之后再更新一些其他的属性值


再然后是 `RectChart`，依旧需要 `InitOpts()` 来完成参数的实例化
```
class RectChart(Chart):
    def __init__(self, init_opts: types.Init = opts.InitOpts()):
        super().__init__(init_opts=init_opts)
        self.options.update(xAxis=[opts.AxisOpts().opts], yAxis=[opts.AxisOpts().opts])
```
这里还出现了新的类 `AxisOpts()`，`class AxisOpts(BasicOpts)` 它同样继承于 `BasicOpts`，所以它是一个**只被当作数据结构而频繁使用的类（重要的话需要重复多遍！）**  


最后就是最外层的 `Bar`，只定义了一个方法 `add_yaxis`，官方注释
> Bar chart presents categorical data with rectangular bars with heights or lengths proportional to the values that they represent  
> 柱状图用矩形条表示分类数据，矩形条的高度或长度与它们所代表的值成正比  

到目前为止，成功完成了 `bar = Bar()`，获取到实例化对象 `bar`



# 设置图数据
接下来 `add_xaxis` 设定了 X 轴的值，传入的序列（Sequence）也是存放在了 `options["xAxis"][0]` 和 `_xaxis_data` 属性中
```
def add_xaxis(self, xaxis_data: Sequence):
    self.options["xAxis"][0].update(data=xaxis_data)
    self._xaxis_data = xaxis_data
    return self
``` 

然后是设定 Y 轴的值，在 `add_yaxis` 中，先通过 `_append_color` 更新了 `colors` 和 `options` 的值
```
def _append_color(self, color: Optional[str]):
    if color:
        self.colors = [color] + self.colors
        if self.theme == ThemeType.WHITE:
            self.options.update(color=self.colors)
```

再通过 `_append_legend` 更新了 `option["legend"]` 的值，目前是 `{'data': ['商家A'], 'selected': {'商家A': True}}`
```
def _append_legend(self, name, is_selected):
    self.options.get("legend")[0].get("data").append(name)
    self.options.get("legend")[0].get("selected").update({name: is_selected})
```
最后再将所有的数据以字典的形式追加到 `options["series"]` 中，现在它是一个字典列表



# 生成 html
完成了所有的数据变更后就需要生成 ECharts 图了，看一下 `render()` 方法  
```
def render(
    self,
    path: str = "render.html",
    template_name: str = "simple_chart.html",
    env: Optional[Environment] = None,
    **kwargs,
) -> str:
    self._prepare_render()
    return engine.render(self, path, template_name, env, **kwargs)
```

`render()` 被定义在了 `Base` 类中，首先执行 `_prepare_render()`
```
def _prepare_render(self):
    self.json_contents = self.dump_options()
    self._use_theme()


def dump_options(self) -> str:
    return utils.replace_placeholder(
        json.dumps(self.get_options(), indent=4, default=default, ignore_nan=True)
    )


def replace_placeholder(html: str) -> str:
    return re.sub('"?--x_x--0_0--"?', "", html)


def get_options(self) -> dict:
    return utils.remove_key_with_none_value(self.options)


def remove_key_with_none_value(incoming_dict):
    if isinstance(incoming_dict, dict):
        return _expand(_clean_dict(incoming_dict))
    elif incoming_dict:
        return incoming_dict
    else:
        return None


def _expand(dict_generator):
    return dict(list(dict_generator))


def _clean_dict(mydict):
    for key, value in mydict.items():
        if value is not None:
            if isinstance(value, dict):
                value = _expand(_clean_dict(value))

            elif isinstance(value, (list, tuple, set)):
                value = list(_clean_array(value))

            elif isinstance(value, str) and not value:
                # delete key with empty string
                continue

            yield key, value


def _clean_array(myarray):
    for value in myarray:
        if isinstance(value, dict):
            yield _expand(_clean_dict(value))

        elif isinstance(value, (list, tuple, set)):
            yield list(_clean_array(value))

        else:
            yield value
```

先从最底层的 `_clean_dict` & `_clean_array` 看起，这两个都是生成器  

- `_clean_dict` 遍历字典并返回所有的 kv，如果 v 是字典，就作为参数继续递归执行 `_clean_dict`，如果是列表、元组、集合，就交给 `_clean_array` 去处理，如果是空字符串就直接跳过本条，其他情况直接返回  

- `_clean_array` 遍历序列所有元素，如果是字典就交给 `_clean_dict` 处理，如果是列表、元组、集合，就作为参数继续递归执行 `_clean_array` 并将生成器的所有值存入列表，其他情况直接返回 

可以发现，所有的 `_clean_dict` 外面都有一层 `expand()`，其将一个生成器的所有元素先存入列表（元组列表），再转换成字典。举个例子帮助理解
```
def test():
    for i in range(5):
        yield i, i

a = list(test()) 
print(a)  # [(0, 0), (1, 1), (2, 2), (3, 3), (4, 4)]
print(dict(a))  # {0: 0, 1: 1, 2: 2, 3: 3, 4: 4}
```

当这三个函数组合起来在 `remove_key_with_none_value` 中通过 `_expand(_clean_dict())` 调用时，功能就是：  
1. 将所有的 value 为空字符串的键值对去除  
2. 将所有的 value 为元组、集合的都转为列表
3. 最后获得处理完成的 `options` 字典  

对这个逻辑表示怀疑的话，可以将三个函数单独的提取出来，执行下面的测试代码
```
test = {"a": {1: "", 2: 2, 3: (1, 2)}, "b": {2}, "c": ["c"]}
print(_expand(_clean_dict(test)))  # {'a': {2: 2, 3: [1, 2]}, 'b': [2], 'c': ['c']}
```

理清这块的逻辑，我们回到 `dump_options`，将处理好的字典通过 `json.dumps()` 转成 json 字符串（集合是不能转的）。再通过 `replace_placeholder` 中的 `re.sub('"?--x_x--0_0--"?', "", html)` 将 `"--x_x--0_0--"` 去除（这个字符串只在 `JsCode` 类的 `js_code` 属性拼接出现过）  

`_prepare_render` 的逻辑结束后就是 `_use_theme` 对属性 `js_dependencies` 处理  


最后回到 `engine.render(self, path, template_name, env, **kwargs)`
```
def render(
    chart, path: str, template_name: str, env: Optional[Environment], **kwargs
) -> str:
    RenderEngine(env).render_chart_to_file(
        template_name=template_name, chart=chart, path=path, **kwargs
    )
    return os.path.abspath(path)
```

还是先看 `RenderEngine` 类
```
class RenderEngine:
    def __init__(self, env: Optional[Environment] = None):
        self.env = env or CurrentConfig.GLOBAL_ENV
```

如果 `env` 为 False （False, None, 0, [] 等都是 False），则 `self.env` 为 `CurrentConfig.GLOBAL_ENV` 否则为 `env` （学到了这样的写法）
`CurrentConfig` 类只有几个常量，`GLOBAL_ENV` 用到了模板引擎 `jinja2`，通过相对路径，载入了 `pyecharts/render/templates` 路径下的模板
```
class _CurrentConfig:
    PAGE_TITLE = "Awesome-pyecharts"
    ONLINE_HOST = OnlineHostType.DEFAULT_HOST
    NOTEBOOK_TYPE = NotebookType.JUPYTER_NOTEBOOK
    GLOBAL_ENV = Environment(
        keep_trailing_newline=True,
        trim_blocks=True,
        lstrip_blocks=True,
        loader=FileSystemLoader(
            os.path.join(
                os.path.abspath(os.path.dirname(__file__)), "render", "templates"
            )
        ),
    )
```

回到 `render` 看 `RenderEngine(env).render_chart_to_file(template_name=template_name, chart=chart, path=path, **kwargs)`
```
def render_chart_to_file(self, template_name: str, chart: Any, path: str, **kwargs):
    """
    Render a chart or page to local html files.

    :param chart: A Chart or Page object
    :param path: The destination file which the html code write to
    :param template_name: The name of template file.
    """
    tpl = self.env.get_template(template_name)
    html = utils.replace_placeholder(
        tpl.render(chart=self.generate_js_link(chart), **kwargs)
    )
    write_utf8_html_file(path, html)


```
默认选择 `pyecharts/render/templates/simple_chart.html` 作为模板  
通过 `generate_js_link()` 生成所需 `chart` 参数
```
    @staticmethod
    def generate_js_link(chart: Any) -> Any:
        if not chart.js_host:
            chart.js_host = CurrentConfig.ONLINE_HOST
        links = []
        for dep in chart.js_dependencies.items:
            # TODO: if?
            if dep.startswith("https://api.map.baidu.com"):
                links.append(dep)
            if dep in FILENAMES:
                f, ext = FILENAMES[dep]
                links.append("{}{}.{}".format(chart.js_host, f, ext))
            else:
                for url, files in EXTRA.items():
                    if dep in files:
                        f, ext = files[dep]
                        links.append("{}{}.{}".format(url, f, ext))
                        break
        chart.dependencies = links
        return chart
```
注意，这里的 `chart` 是 `bar` 实例对象。这个静态方法的内部都和 js 有关，先跳过  

通过 `utils.replace_placeholder(tpl.render(chart=self.generate_js_link(chart), **kwargs))` 将模板对象转成的字符串中的指定字符去除
```
def replace_placeholder(html: str) -> str:
    return re.sub('"?--x_x--0_0--"?', "", html)
```

再将其写入 html 文件
```
def write_utf8_html_file(file_name: str, html_content: str):
    with open(file_name, "w+", encoding="utf-8") as html_file:
        html_file.write(html_content)
```
最后返回生成的 html 文件路径，至此，整个代码逻辑结束
