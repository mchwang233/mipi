# CSI-2 要点汇总
## 摘要
CSI-2（Camera Serial Interface 2）主要是定义一种设备（主要是camera）与移动处理器直接的一种接口。
## 典型形态
![image](https://github.com/user-attachments/assets/86944a56-817f-4a58-8045-0e20f05feb54)
![image](https://github.com/user-attachments/assets/d7bd7a9d-e733-4943-ab53-1246e756c122)
## 层级定义
![image](https://github.com/user-attachments/assets/07095a7b-ba86-4a04-a8cb-21e686fc263d)
### PHY layer
物理层，主要理解为实现模拟信号采集处理传输，具体表现形式为CPHY，DPHY等。
### protocol layer
协议层，主要包含一些pixel的处理，协议底层实现，lane管理。
### Application layer
应用层。
