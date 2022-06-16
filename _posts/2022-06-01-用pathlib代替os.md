---
title: 用 pathlib 代替 os
date: 2022-06-01
updated: 2022-06-02
categories: 
- Python
tags:
- os
- pathlib
---


在 StackOverflow 上回答文件路径相关的问题时用到了 `os.path`，有人在 comment 里面提到 `pathlib` 更为强大  
之前只知道 `subprocess` 可以用来代替 `os.system` 等等，于是翻了一下官方文档，先附上链接 [pathlib — Object-oriented filesystem paths](https://docs.python.org/3/library/pathlib.html)  
本文主要记录平时常用的 `os` 功能用 `pathlib` 替代  
`os` 和 `pathlib` 的映射表放在了文末，需要的请[前往文末查看](#end)


# 基础用法
相较 `os` 和 `os.path` 的函数式，pathlib 采用了面向对象式的设计模式，本节点为官方文档中的 **base use**
## 获取所有子目录列表
```
from pathlib import Path

p = Path(".")
[x for x in p.iterdir() if x.is_dir()]
```

## 目录树中的指定文件
```
# 获取所有 py 文件
list(p.glob("**/*.py"))  # generator
```

## 在目录树中导航
```
q = p / "test" / "test.file"
q.resolve()
```

## 查询路径属性
```
q.exists()
q.is_dir()
```

## 打开一个文件
```
with q.open() as f:
    f.readline()
```


# 用法对比
本节仅介绍部分用法，详情请参考官方文档
## 绝对路径
os
```
# 相对于当前工作目录的绝对路径
os.path.abspath("test.file")
```
Path
```
# 相对于当前工作目录的绝对路径
Path("test.file").absolute()
# 使用根目录符号开头，不再涉及当前工作目录
Path("/test.file").absolute()

# resolve 将路径或路径片段的序列解析为绝对路径，并且会解析 .. 符号
Path("test/../test.file").resolve()
```

## 路径拼接
os
```
os.path.join("/", "test", "test.file")
```
Path
```
Path("/").joinpath("test", "test.file")

Path("/") / Path("test") / Path("test.file")
Path("/") / Path("test") / "test.file"
"/" / Path("test") / "test.file"
```
**path 和 os 相同，如果拼接的路径是根符号 / 开头，则前面的路径将被忽略**

## 创建目录
os
```
os.mkdir("test")
# 根据结构创建目录
os.makedirs("test/test")
```
Path
```
# parents 父目录不存在，是否创建父目录
# exist_ok 目录存在时不创建目录也不抛异常
Path("test/test").mkdir(parents=True, exist_ok=True)
```

## 当前工作目录
os
```
os.getcwd()
```
Path
```
Path.cwd()
```


## 切换工作目录 
os
```
os.chdir("test")
```
Path 不支持切换工作目录，可以转为 `os.chdir()` 的参数
```
os.chdir(Path("test"))
```


## 获取目录下文件
os
```
os.listdir()  # list
```
Path
```
Path().iterdir()  $ generator
```


## 类型判断
os
```
os.path.isdir("test")
os.path.isfile("test")
os.path.exists("test")
```
Path
```
a = Path("test").is_dir()
b = Path("test").is_file()
c = Path("test").exists()
```


## 获取文件/目录名和文件后缀  
os
```
os.path.basename("/test/test.file")
# 如果要获取后缀需要 split 切割
```
Path
```
# 名称 + 后缀
Path("/test/test.file").name  # test.file

# 仅文件
Path("/test/test.file").stem  # test

# 后缀
Path("/test/test.file").suffix  # .file
Path("/test/test.file.txt").suffix  # .txt
Path("/test/test.file.txt").suffixes  # ['.file', '.txt']
```


## 获取父目录路径
os
```
os.path.dirname(__file__)
```
Path
```
# 父目录
Path(__file__).parent
Path(__file__).parents[0]

# 父目录的父目录
Path(__file__).parent.parent
Path(__file__).parents[1]
```


## 获取 Home 目录
os
```
os.path.expanduser('~')
```
Path
```
Path.home()
Path("~").expanduser()
```


## 创建文件
Python 没有内置的创建文件方法（比如 linux 的 touch）  
现在可以用 Path 的 touch
```
Path("test.file").touch()
```


## 操作文件
Path 对象自带操作文件方法
```
# 写入
Path("test.file").write_text("CN-LanBao\n")

# 读取
Path("test.file").read_text()

# 获取文件句柄
with Path("test.file").open() as f:
    for line in f:
        print(line)
```
注意 `Path` 多次写入，会覆盖原有数据。读取也没用缓冲区，使用的时候需要注意


## 生成修改后的文件名或路径后缀
os 需要 split 切割后字符串拼接，这里不展示  
Path
```
Path("test/test.file").with_name("test.txt")  # test/test.txt

# ubuntu 会报错 AttributeError: 'PosixPath' object has no attribute 'with_stem'
Path("test/test.file").with_stem("t")  # test\t.file

Path("test/test.file").with_suffix(".file.txt")  # test\test.file.txt
```


# os 和 pathlib 中的等价映射表  <span id="end"></span>
> 并不是所有的函数/方法都是等价的，尽管有一些重叠的用例，但是有不同的语义。包括 `os.path.abspath()` 和 `Path.resolve()`，`os.path.relpath()` 和 `PurePath.relative_to()`  

|os and os.path|pathlib|
|--|--|
|os.path.abspath()|Path.resolve()|
|os.chmod()|Path.chmod()|
|os.mkdir()|Path.mkdir()|
|os.makedirs()|Path.mkdir()|
|os.rename()|Path.rename()|
|os.replace()|Path.replace()|
|os.rmdir()|Path.rmdir()|
|os.remove(), os.unlink()|Path.unlink()|
|os.getcwd()|Path.cwd()|
|os.path.exists()|Path.exists()|
|os.path.expanduser()|Path.expanduser() and Path.home()|
|os.listdir()|Path.iterdir()|
|os.path.isdir()|Path.is_dir()|
|os.path.isfile()|Path.is_file()|
|os.path.islink()|Path.is_symlink()|
|os.link()|Path.hardlink_to()|
|os.symlink()|Path.symlink_to()|
|os.readlink()|Path.readlink()|
|os.path.relpath()|Path.relative_to()|
|os.stat()|Path.stat(), Path.owner(), Path.group()|
|os.path.isabs()|PurePath.is_absolute()|
|os.path.join()|PurePath.joinpath()|
|os.path.basename()|PurePath.name|
|os.path.dirname()|PurePath.parent|
|os.path.samefile()|Path.samefile()|
|os.path.splitext()|PurePath.suffix|