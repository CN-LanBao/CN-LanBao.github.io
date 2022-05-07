---
title: uiautomator2 二次开发
date: 2022-05-07
categories: 
- Python
tags:
- uiautomator2
---


# 背景
选择 uiautomator2 作为 UI 测试框架，想要实现一些定制化的功能，所以需要对其进行二次开发  
> Github 地址 [uiautomator2](https://github.com/openatx/uiautomator2)  


# 设计思路
本文涉及的 log 相关操作，使用的是自己用 logging 封装的 log 类，详情请参考 [logging 实现日志类](https://cn-lanbao.github.io/python/2022/05/05/logging%E5%AE%9E%E7%8E%B0%E6%97%A5%E5%BF%97%E7%B1%BB/)  
一个从未接触过 uiautomator2 的测试同学想要将其用起来，至少要先完成库安装，然后通过度娘或 github 等其他途径了解到需要完成前置操作 init 和 connect。所以，写了一个 Uiautomator 类，在 `__init__` 中完成 uiautomator2 init 和 device connect 操作。使用时，仅需要对 Uiautomator 类进行实例化即可
```
class Uiautomator(object):

    def __init__(self, device_id):
        self.device_id = device_id
        # 为设备创建 logger 对象
        self.log_util = LogUtil(self.device_id, logger_name=self.device_id)
        # uiautomator2 初始化
        try:
            init_cmd = "python -m uiautomator2 init {}".format(self.device_id)
            self.log_util.info(init_cmd)
            init_process = subprocess.Popen(init_cmd.split(), stdout=subprocess.PIPE, stderr=subprocess.STDOUT)
            while init_process.poll() is None:
                line = init_process.stdout.readline()
                if line:
                    self.log_util.debug(line)
            # 被测设备
            self.dut = u2.connect(self.device_id)
        except Exception:
            msg = "Fail to connect uiautomator2: \n{}".format(traceback.format_exc())
            self.log_util.error(msg)
            raise UiautomatorException(msg)
```
在 Uiautomator 类中重新定义了常用的方法，补全了注释和参数的影响范围。像 resourceId、text、xpath 实例化控件对象的不同写法 `d.xpath(xpath)` 和 `d(text=text)`，也通过定义不同方法来统一
```
@u2_log
def click_resource_id(self, resource_id):
    """
    点击指定 resource id 控件
    @Author: CN-LanBao
    @Create: 2022/5/7 14:13
    :param resource_id: 控件 resourceId 属性值
    :return: None
    """
    return self.dut(resourceId=resource_id).click()

@u2_log
def click_text(self, text):
    """
    点击指定 text 控件
    @Author: CN-LanBao
    @Create: 2022/5/7 14:15
    :param text: 控件 text 属性值
    :return: None
    """
    return self.dut(text=text).click()

@u2_log
def click_xpath(self, xpath):
    """
    点击指定 xpath 控件
    @Author: CN-LanBao
    @Create: 2022/5/7 14:16
    :param xpath: 控件 xpath 属性值
    :return: None
    """
    return self.dut.xpath(xpath).click()
```