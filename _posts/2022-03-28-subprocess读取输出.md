---
title: subprocess 读取输出
date: 2022-03-28
categories: 
- Python
tags:
- subprocess
- os
---


执行 subprocess.Popen 时，需要持续读取其输出（subprocess.run 的读取方法不做赘述）  
以 `ping www.baidu.com -c3` 为例，通过 `iter()` 创建一个迭代器，实时读取执行时的标准输出

```
import subprocess

cmd = "ping www.baidu.com -c3"
ping_pipe = subprocess.Popen(cmd, shell=True, stdout=subprocess.PIPE, universal_newlines=True)
for info in iter(ping_pipe.stdout.readline, ""):
    print(info)
print("END")
```

以上的代码就能实现读取标准输出的需求了。接下来把需求升级一下，去掉命令中的 `-c3` 参数，如果标准输出中出现了指定的关键字，或运行了超过 5s，都不再读取输出  
我们可以定义一个时间变量来控制运行时间，代码如下

```
import time
import subprocess

cmd = "ping www.baidu.com"
ping_pipe = subprocess.Popen(cmd, shell=True, stdout=subprocess.PIPE, universal_newlines=True)
start_time = time.time()
for info in iter(ping_pipe.stdout.readline, ""):
    print(info)
    if "keyword" in info:
        break
    if time.time() - start_time > 5:
        print("Timeout!")
        break
print("END")
```

以上代码在处理这种**持续有输出的命令**时已经足够了，但其中隐藏了一个问题，我们将命令修改为 `ping www.baidu.com | grep nonexistent` 再运行上面的代码  
程序并没有按照我们预期的在执行 5s之后打印 `Timeout!` 和 `END` 并退出。因为该条命令不会有任何的输出（grep 的参数是一个不存在的字符串），而 `readline` 读取的时候是**阻塞**的（若没有信息则一直等待，直到读取到信息），所以在第一轮迭代的时候就在 `readline` 阻塞住了  
该问题在 Pycharm 等 idea 中可以通过 debug 模式直观的发现  
解决该问题，一种思路是启动一个线程来承载读取的逻辑，主进程只要在需要的时候判断一下线程的状态即可。这里主要介绍第二种方法，所以不在此对线程的代码进行展示  
既然是 `readline` 的阻塞读取导致的，那么能否将其改为非阻塞（无论有无信息，皆继续执行）读呢？这里用到了 `fcntl` 库，代码如下

```
import os
import time
import fcntl
import subprocess

cmd = "ping www.baidu.com | grep nonexistent"
ping_pipe = subprocess.Popen(cmd, shell=True, stdout=subprocess.PIPE, universal_newlines=True)
start_time = time.time()

fd = ping_pipe.stdout.fileno()
fl = fcntl.fcntl(fd, fcntl.F_GETFL)
fcntl.fcntl(fd, fcntl.F_SETFL, fl | os.O_NONBLOCK)

while ping_pipe.poll() is None:
    info = ping_pipe.stdout.readline()
    if "keyword" in info:
        break
    if time.time() - start_time > 5:
        print("Timeout!")
        break
print("END")
```

但是该方法并不是完美的，具体问题已在 StackOverflow 抛出，感兴趣的话请 follow  
[How to get stdout from subprocess.Popen elegantly with timeout check?](https://stackoverflow.com/questions/71644398/how-to-get-stdout-from-subprocess-popen-elegantly-with-timeout-check)
