---
title: 快速美化PyQt应用--QCandyUi
date: 2018/07/11
categories: 
- 前端
tags:
- python
- Qt
---
#### QCandy-UI
* 快速美化PyQt应用
* 项目地址: [https://github.com/shuoGG1239/QCandyUi](https://github.com/shuoGG1239/QCandyUi)

#### 使用方法
* pip install QCandyUi
* 仅需在需要美化的窗口类上加上@colorful装饰器即可
* 也可以调用CandyWindow.creatWindow()返回经美化的QWidget (推荐用这种)

#### 实例
* 原味窗口
```python
# 窗口类为TcpUdpSerialPortTool
# TcpUdpSerialPortTool.py
class TcpUdpSerialPortTool(QWidget):
    ... ...

# main.py
app = QApplication(sys.argv)
mainWindow = TcpUdpSerialportTool.TcpUdpSerialPortTool()
mainWindow.show()
sys.exit(app.exec_())
```
![norm](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pLmxvbGkubmV0LzIwMTgvMDcvMDQvNWIzYzQwNGJkZTQxZi5wbmc?x-oss-process=image/format,png)
  
  
* 加了蓝绿色主题的窗口(使用@colorful)
```python
# 窗口类为TcpUdpSerialPortTool
# TcpUdpSerialPortTool.py
from QCandyUi.CandyWindow import colorful

@colorful('blueGreen')
class TcpUdpSerialPortTool(QWidget):
    ... ...

# main.py
app = QApplication(sys.argv)
mainWindow = TcpUdpSerialportTool.TcpUdpSerialPortTool()
mainWindow.show()
sys.exit(app.exec_())
```
![blueGreen](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pLmxvbGkubmV0LzIwMTgvMDcvMDQvNWIzYzQxMmJjMjk3Ny5wbmc?x-oss-process=image/format,png)
  
  
* 加了蓝色主题的窗口(使用@colorful)
```python
# 窗口类为TcpUdpSerialPortTool
# TcpUdpSerialPortTool.py
from QCandyUi.CandyWindow import colorful

@colorful('blue')
class TcpUdpSerialPortTool(QWidget):
    ... ...

# main.py
app = QApplication(sys.argv)
mainWindow = TcpUdpSerialportTool.TcpUdpSerialPortTool()
mainWindow.show()
sys.exit(app.exec_())
```
![blue](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pLmxvbGkubmV0LzIwMTgvMDcvMTAvNWI0NDk5MTEwM2E4Ny5wbmc?x-oss-process=image/format,png)
* 加了蓝色主题的窗口(使用CandyWindow.createWindow)
```python
from QCandyUi import CandyWindow

mainWindow = TcpUdpSerialportTool.TcpUdpSerialPortTool()
mainWindow = CandyWindow.createWindow(mainWindow, 'blue')
mainWindow.show()
```

# Ps:
* 想自己新增颜色主题可以在theme.json里面配, 按照theme.json里的格式配即可
* py模块的安装包在/python-version/dist中