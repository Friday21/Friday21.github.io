---
layout:     post
title:      "用Python控制Keithley测量仪器"
date:       2016-06-01
author:     "FridayLi"
catalog: false
tags:
  - Python
---

# 背景
作为一名凝聚态物理学生，做科研的大部分时间都在和各种测量仪器打交道，我最常用的要数Keithley 2400, 2410等测量电信号的仪器了，Keithley仪器的分辨率还是很高的，2400测量电压和电流都能精确到纳(10^-9)的量级，6517更是能达到皮（10^-12）的量级, 非常了不起。实验室用来控制这几台仪器的程序都是Labview程序，属于G语言吧，研一刚进实验室的时候师姐给我讲了一个多小时才给我讲明白一个简单测量IV的程序流程，研一寒假前为了能够实现Labview调用的子程序中的一个参数能够在图表上实时显示，费了老大劲了，虽然最终实现了，但现在基本忘记怎么做的了，总之很复杂。看一下Labview的程序（其实就是画图啦：  

![描述](/img/old-post/8688e1980f7a58fbcf27d7d26ebeab2c.PNG)  
![描述](/img/old-post/80ac9e3a40da74800508707f2fc7f9eb.PNG)   

前面板UI还好，但后面板程序图真是太不具有可读性啦，扩展性也很差，想添加个新功能得画半天图，于是我想如果能用Python控制这些测量仪器就好啦，这样就可以把每个测量功能封装成一个函数，需要扩展新功能的话直接调用再修改就好啦，抱着试试看的态度（买了一疗程）在github上搜索Keithley真的搜出来几条Python写的控制程序，好开心，于是下定决心把自己常用的几个Labview测量程序Python化！

# 1. 环境搭建
首先是接口的连接，要通过Python连接上GPIB接口需要对应的库，这里用到的是pyvisa, 官方教程在[这里](https://pyvisa.readthedocs.org/en/stable/), 我用的Python IDE是pycharm，所以直接在pycharm上搜索安装pyvisa就好了（真的很方便），但根据pyvisa的说明还需要安装 National Instruments’s VISA ，去[官网]("http://search.ni.com/nisearch/app/main/p/ap/tech/lang/zhs/pg/1/sn/ssnav:ndr/fil/AND(nicontenttype:driver,%20sitesection:ndr,%20AND%20(OR(nigen10:1640,%20productcategories:1640,%20%22NI-VISA%22)%20,%20OR(nilanguage:zh-CN,%20nilanguage:en)")下载适合自己电脑的版本，由于目前Linux平台只支持32位的，而我的是Ubuntu 14.04LTS，没办法，只好装在win10上了，pyvisa和National Instruments’s VISA都安装成功后就可以进行下一步了  


# 2. 连接仪器
如果用的台式机，又有GPIB扩展槽，直接连上仪器就行了，我用的笔记本，所以还需要一条USB-GPIB线，这里用的是KUSB-488A  ，第一次用肯定要装驱动的，若不能自动安装则需要去官网下载驱动，一切就绪后，执行以下Python语句以检测是否成功连上仪器：
```
import visa
rm = visa.ResourceManager()
rm.list_resources()
('ASRL1::INSTR', 'ASRL2::INSTR', 'GPIB0::12::INSTR')
inst = rm.open_resource('GPIB0::12::INSTR')
print(inst.query('*IDN?'))
```
若果第三句执行后能找到你的仪器则大功告成，否则就要检查哪个驱动出了问题
# 3. 简单IV测量程序
* 首先连接仪器
```python
# 默认GPIB_address 地址为17（Keithley 2410）,2400为15
GPIB_address = 15
def connect_inst():
    rm = visa.ResourceManager()
    inst = rm.open_resource('GPIB0::%d::INSTR' % GPIB_address)
    inst.write(':outp on')
    return inst
```
其中像`:output on`之类的命令是根据Keithley 2400仪表的说明书来编写的，各个功能的实现都要参考说明书上的命令集
* 测量结束后关闭仪器，使参数恢复初始状态, 关闭输出
```python
def close_inst(inst):
    inst.write("*RST")
    inst.write("*CLS")
    inst.write("SYSTEM:TIME:RESET:AUTO 0")
    inst.write(':outp off')
```
* 测量过程
```python
def IV_sweep(start=-3, end=3, step=60, delay=100, interval=300):
    '''
    :param start: 起始测量电压
    :param end: 结束测量电压
    :param step: 测量分多少步进行
    :param delay: 测量延迟（ms）
    :param interval: 测量间隔（ms）
    '''
    inst = connect_inst()
    # 使电压量程自动随输入电压的值变化
    Range = 1.1*(math.fabs(end) if math.fabs(end) > math.fabs(start) else math.fabs(start))
    inst.write(':SOUR:VOLT:RANG ' + str(Range))
    inst.write(':SOUR:DEL '+str(delay/1000))
    stage = (end - start)/step
    Ilist = list()
    Vlist = list()
    for i in range(step+1):
        V = start + stage*i
        # 设置测量电压
        inst.write(':source:volt %s' % V)
        # 读取参数
        inst.write('read?')
        data = inst.read("TRACE:DATA")
        I = float(data.split(',')[1])
        # 将测量结果保存到Ilist 和Vlist中
        Ilist.append(I)
        Vlist.append(V)
        inst.write(':source:volt 0')
        time.sleep(interval/1000)
    close_inst(inst)
```
这样测量结果就保存在两个list当中了，这当然不是我们要的最终结果，还需要把测量到的数据保存到文件中，而且测量过程中要实时绘图，保存数据到文件中比较基本，就不多说了，可以参考我上传到github上的[代码](https://github.com/Friday21/Keithley_measure)，接下来说一说在Python下怎么实现动态绘图
# 4. matplotlib实时绘图
Python有一个很好的绘图的第三方库——**matplotlib**, 可以实现和matlab相媲美的绘图功能，相当好用。
* 用matplotlib简单的绘图
```python
import matplotlib.pyplot as plt
plt.plot([1,2,3,4], [1,4,9,16], 'r-o')
plt.axis([0, 6, 0, 20])
plt.ylabel('current')    #为y轴加注释
plt.show()
```
效果如下：  

![描述](/img/old-post/fd949d4d7d66c09bcab30c76f53a77b6.PNG)   

* 实现实时绘图
若要实现实时绘图，就要开启plot的交互模式：
```python
  # 开启实时绘图
    plt.ion()
    for i in range(step+1):
        ......
        plt.axis([min(Vlist)*1.1, max(Vlist)*1.1, min(Ilist), max(Ilist)])
        plt.plot(Vlist, Ilist, r'b-D')
        # 这个为停顿0.01s，能得到产生实时的效果
        plt.pause(0.1)
        if i == step:
            # 保存图片
            savefig(savefile.name[:-4]+'.png')
            # 关闭交互模式
            plt.ioff()
            plt.close()
            savefile.close()
```
测试结果如下图：  
![描述](/img/old-post/98c32695b8e68a6886609b285d22792d.PNG)	  


再加上保存数据到文件的功能后就可以实现和Labview同样的功能啊，看了下代码，100行，虽然也不短，但逻辑很清楚，以后便于修改，下面一个例子将充分体现Python对Labview的优势。  
# 5. 温度控制
其实不仅仅是Keithley，其它GPIB连接的仪器也可以同样用Python控制，比如温度控制仪model 331  
下面是一个温度控制的小程序：
```python
def term_ctrl(start, end, speed):
    '''
    :param start: 起始温度  K
    :param end: 目标温度    K
    :param speed: 变温速度  K/min
    :return:
    '''
    inst = connect_inst()
    inst.write('ramp 1,0')
    time.sleep(0.5)
    inst.write('setp 1,' + str(start))
    time.sleep(0.5)
    inst.write('ramp 1,1,' + str(speed))
    time.sleep(0.5)
    inst.write('setp 1,' + str(end))
```
短短几行就可以实现Labview的温控程序啦，方便不只是一点点！而且扩展起来很方便，比如：  
```python
term_ctrl(290, 240, 1)
time.sleep(50*60)
term_ctrl(240, 220, 0.3)
time.sleep(70*60)
term_ctrl(220, 170, 0.5)
```
这样短短几行就可以实现在不同阶段以不同速率降温啦！

# 6. 参考文章
* [pyvisa 官方文档](https://pyvisa.readthedocs.org/en/stable/)
* [matplotlib官方文档](http://matplotlib.org/1.5.1/users/pyplot_tutorial.html)  
* [matplotlib 实时绘图](http://www.noneface.com/2015/10/25/EE-python-serial.html)
* [github上的Keithley程序](https://github.com/hinnefe2/keithley)  

本文讲到的程序可以在[这里](https://github.com/Friday21/Keithley_measure)找到源代码