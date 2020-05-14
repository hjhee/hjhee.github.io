+++
title = "Introduction to Qt Quick Development"
date = 2016-06-28T13:57:47+08:00
tags = ["qml"]
draft = false
+++

简易开发文档

<!--more-->

# Qt Quick 介绍

[Qt Quick](http://doc.qt.io/qt-5/qtquick-index.html)

Qt Quick provides everything needed to create a rich application with a fluid and dynamic user interface. It enables user interfaces to be built around the behavior of user interface components and how they connect with one another, and it provides a visual canvas with its own coordinate system and rendering engine. Animation and transition effects are a first class concept in Qt Quick, and visual effects can be supplemented through specialized components for particle and shader effects.

## Qt 开发环境

[Qt在Windows下安装](https://lug.ustc.edu.cn/sites/qtguide/ch01-03.htm)

选择Qt 5.6，MSVC 2015
库至少包括：

*   Qt Quick Controls
*   Qt WebEngine
*   Qt Labs Controls(TP)

## Qt 插件

使用了如下插件：

[QuickPlot](http://www.arnorehn.de/blog/2014/12/quickplot-a-collection-of-native-qtquick-plot-items/): A collection of native QtQuick plotting items
用于DA输出设置的绘图，用git下载源代码编译安装。

[QtXlsxWriter](https://github.com/VSRonin/QtXlsxWriter): .xlsx file reader and writer for Qt5
用于从数据库导出Excel表格

[Qt Charts](https://blog.qt.io/blog/2016/01/18/qt-charts-2-1-0-release/): 用于波形图绘制
[Qt Charts安装教程](http://blog.csdn.net/onlyshi/article/details/50666858)

# Qt 特性

## qml 语言

QML (Qt Meta-Object Language)，是Qt推出的Qt Quick技术的一部分，是一种新增的简便易学的语言。QML是一种陈述性语言，用来描述一个程序的用户界面。文件格式以.qml结尾。语法格式非常像CSS，但又支持javacript形式的编程控制。在QML，一个用户界面被指定为具有属性的对象树。 这使得Qt更加便于很少或没有编程经验的人使用。 JavaScript在QML中作为一种脚本语言，对QML进行逻辑方面的编程。

## C++/qml 交互

C++与qml交互是重要的一环，Context Properties是一个很好的办法。

[Integrating QML and C++](http://doc.qt.io/qt-5/qtqml-cppintegration-topic.html)
[Exposing Attributes of C++ Classes to QML](http://doc.qt.io/qt-5/qtqml-cppintegration-exposecppattributes.html)
[Embedding C++ Objects into QML with Context Properties](http://doc.qt.io/qt-5/qtqml-cppintegration-contextproperties.html)
[Data Type Conversion Between QML and C++](http://doc.qt.io/qt-5/qtqml-cppintegration-data.html)

## QThread

关于Qt的多线程使用，应参考如下文章：

[QThread Class](http://doc.qt.io/qt-5/qthread.html)
[How to use QThread in the right way (Part 1)](http://blog.debao.me/2013/08/how-to-use-qthread-in-the-right-way-part-1/)
[QThread: You were not doing so wrong.](https://woboq.com/blog/qthread-you-were-not-doing-so-wrong.html)
[what is the correct way to implement a QThread… (example please…)](http://stackoverflow.com/questions/4093159/what-is-the-correct-way-to-implement-a-qthread-example-please)
[How To Really, Truly Use QThreads; The Full Explanation](https://mayaposch.wordpress.com/2011/11/01/how-to-really-truly-use-qthreads-the-full-explanation/)

## WebEngineView

在QML界面中嵌入网页，利用js库可以绘制漂亮的图表，如[flotcharts](http://www.flotcharts.org/)。新一代Qt的浏览器引擎是QWebEngine，由此引入了C++/qml/HTML之间通信的问题。

In Qt5.6, if you want to make C++ part and JavaScript to communicate, the only way to do it is using QWebChannel on a QWebEngineView.

[How to use Qt WebEngine and QWebChannel?](http://stackoverflow.com/questions/28565254/how-to-use-qt-webengine-and-qwebchannel)
[实现QT与HTML页面通信](http://blog.csdn.net/liuyez123/article/details/50509788)
[QT WebEngineView Communication with Javascript](http://blog.trumpton.org.uk/2015/09/qt-webengineview-communication-with.html)

# 移动式气体检测仪

设计思想见论文和PPT，程序主要有如下功能:

*   设备控制
*   操控界面
*   数据记录
*   数据上传

利用USB2089驱动控制设备，函数用法见阿尔泰文档。需要导入驱动库(USB2089_32.lib)到Qt Creator项目，见教程：[添加外部库导入Qt Creator项目](http://qa.helplib.com/489803)。

Device类提供C++接口给QML，内部初始化了DAQ和DAC类，分别用于数据采集和波形生成。采集数据之间通过全局数组qint16 adBuffer[1024]共享数据，定义于main.cpp。

Device类有若干成员：
*   m_gasData: 将电压数据转换为气体浓度，数据暴露给QML
*   m_gasTimer： 气体浓度数据更新定时
*   m_serial: 串口控制相关
*   m_daPara: QML界面设置的DA参数保存于此
*   m_dio: 设置电磁阀/气泵的DO，包括温控PID的PWM生成
*   m_devTimer: 设备采集时间的计时器
*   m_adSource: 将最新的电压数据放入qt charts的series中, m\_adSource提供给QML接口
*   m\_daSource: 与m\_adSource同理

Q_PROPERTY相关见[Embedding C++ Objects into QML with Context Properties](http://doc.qt.io/qt-5/qtqml-cppintegration-contextproperties.html)。

Device的m\_status反应了采集状态，0为停止，1为采集。m\_status通过Q_PROPERTY暴露给QML。

DAQ负责数据采集，启用独立线程工作，同时设定了定时器，重写了timerEvent()。关于timerEvent()见[Qtimer vs timerEvent - which of them produces less overhead?](http://stackoverflow.com/questions/12628343/qtimer-vs-timerevent-which-of-them-produces-less-overhead)。
DAQ定时器的工作状态受Device的m_status影响。DAQ采集的数据保存在ADBuffer数组内，同时发出信号acquired(QDateTime)。

DAC构造函数定义了通道号(m\_channel)，共12个DAC开辟了12个线程，DAC与DAQ类似，启用timerEvent()定时产生波形，定时周期为20ms。波形参数由Device的m_daPara指定，包括波形、周期、占空比和幅值。
DAC内部维护了相位，定义是m_phase，相位指定了波形产生的阶段。

DAParameter的parameters属性利用了QVariantList，是一个二维js数组，第一维是通道号，第二维是参数值。

DataSource定义了用于Qt Charts绘图的数据源，其设计参考了Qt Creator的例子[Qml Oscilloscope](http://doc.qt.io/qt-5/qtcharts-qmloscilloscope-example.html)，可在屏幕上以60Hz绘制20000点的波形，效率极高。界面显示相关部分也一并参考。

GasData定义了槽： newData()，接收DAQ的信号acquired()，从ADBuffer数组中获取电压数据转换浓度，发出浓度变量数值改变的信号timeChanged(), so2Changed(), no2Changed(), etc... ，通知QML更新显示界面。

Serial负责GPRS通信，波特率115200，开辟线程处理通信事务，通信协议是论文的简化版，有帧头、ID和消息类型和负载。根据消息类型判断负载长度。类型0为心跳包，1为浓度数据上传，2为设备状态返回。当数据到达的时候，信号readyRead()发出，Serial的槽handleReadRead()负责读取数据，newPacket()负责对数据解析。

Database类负责处理数据库事务。DAQ发出acquired()信号之后，Database的槽insert()动作，将数据插入表内。表的结构见PPT，表名通过m_tableName获取。
表名模板为tableyyyy\_mm\_dd\_HH\_MM\_SS\_XXX，可通过界面调整表头为上次值或者是新建表tableSwitch()将更改表名名称。
目前数据库采用PostgreSQL 9.5， Qt采用QPSQL驱动，需要手动添加驱动。教程见[Qt 5.3 MinGW + PostgreSQL 9. Make SQL driver and database connectiion on Windows 7](https://www.youtube.com/watch?v=fBgJ9Azm_S0)
HostName="localhost", DatabaseName="sensor", UserName="postgres", Password="Hallo", Port=5432.
数据库支持插入数据、表名查询和导出操作，这些工作分别交给线程DBWorker完成：开辟线程，初始化DBWorker,通过信号槽将任务分配给工作线程。待线程完成工作后发出信号给Database。

Timer是一个自定义定时器，与QTimer别无二致，但是增加了槽onStatusChanged()，与Device的status连接(connect)可以联动。

QML部分定义的文件需要加入资源文件中(resource.qrc)，否则编译时会报错。文件目录"qrc:/"为项目根目录。QML文件与HTML代码类似，不再描述。