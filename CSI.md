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
## LLP
LLP层通常会处理两种格式的包(Packet)：长包(Long Packet)和短包(Short Packet)。无论哪一种包，LLP层都会根据CSI-2协议的规定，给它们添加包头(Packet Header, PH)和包尾(Packet Footer, PF)，并作为有效数据在HS模式下传输。
同时，LLP层会在每次退出LP后添加一个SoT(Start of Transmission)序列作为开始进入HS模式的标志，并在进入LP模式前添加一个EoT(End of Transmission)序列作为退出HS模式的标志。如下图所示。
![image](https://github.com/user-attachments/assets/932ff24f-1474-455e-add6-8dd8675c91dd)
### 长包结构
#### D-PHY
如果物理层选用D-PHY时，则长包中主要包括三个部分：32bit的PH、一定长度的packet data和8bit的PF。具体结构如下图。
![image](https://github.com/user-attachments/assets/fc3ed1d1-4123-40d2-a2b8-26f34d1452bf)
- 数据标识(DI) ：总共8bit。由VC（2bit）字段和DT（6bit）字段构成，VC字段表示VC id的低两位，DT代表packet layer的数据类型。
- 数据计数(WC) ：总共16bit。从PH结尾到PF起始位置中间的填充数据的长度，也就是packet data的长度。单位为字节。接收端可以通过WC来判断包的结尾位置。
- 虚拟通道拓展+错误校验码(VCX+ECC)：总共8bit。其中，ECC占6bit，采用Hamming Code，可以纠正PH中DI+WC+VCX中的一位错误或发现两位错误；VCX占2bit，代表VC id的高两位，与DI中的VC一起组成4-bit的VC id，总共能表示16个虚拟通道。
- 数据填充(Data Payload，或称为Packet Data) ：长度=WC*8bits。因为WC的存在，接收端可以直接通过WC来判断包尾在什么位置，因此不需要在传输的数据中内嵌标志位等用来同步的信号。也就是说LLP层对具体的数据内容没有任何限制。
- 检验和（CS）：总共16bit。CHECKSUM采用CCITT的16-bit的CRC校检，用来校验packet data中的错误，即x16+x12+x5+x0。CRC只能检测出一个或者多个错误，并不能纠正错误。
#### C-PHY
如果物理层选用C-PHY时，则包括PH、packet data、PF、填充字节FILLER四个部分。具体结构如下图。
![image](https://github.com/user-attachments/assets/2415ce1e-c917-4b03-9849-b775e0d7ab93)
如果C-PHY物理层有N个lane，则PH由以下字段构成
- RES+VCX：总共8 bit，由5-bit RES + 3-bit VCX构成。RES是保留位，暂时没有用处，都置为0。VCX是VC id的最高三位，与DI中的VC字段共同构成5-bit的VC id，总共能代表32个虚拟通道。
- DI：总共8 bit。与D-PHY的DI字段含义相同，包括2bit的VC，代表VC id的低两位；还有6bit的DT，代表packet data的数据类型。
- WC：总共16 bit。与D-PHY的WC字段含义相同，接收端读取WC的值来决定packet data的结尾。
- PH-CRC：总共16 bit。使用CRC校验方式对PH进行校验(RES+VCX+DI+WC)，可以检测出多bit的错误。
- 发送完上述字段后，C-PHY物理层会插入一个同步字(Sync Word)，作为CSI-2 PPI(PHY Protocol Interface)指令的执行结果。
- 然后会再复制一份上述的RES+VCX、DI、WC、PH-CRC的数据放在PH中。因此总共的长度为6×16bits，这是一个lane的PH字段构成。如果物理层有N个lane，则重复N次上述字段，因此PH的总长度为6N×16bits。
PH中的每6个16-bits数据会被广播到N条lane上，这6个word还会继续每3word一组分成两部分，C-PHY物理层会在两者间插入sync word，用于响应CSI-2 Tx的PPI命令。如下图所示。
![image](https://github.com/user-attachments/assets/733aabae-132a-4d34-a8e2-2ee1576808ea)
PF由16 bit的CRC校验计算得到，作用与D-PHY中的相同。

Packet Filler在PF后插入。一般它的值取为0。包含两个作用：

1. PF需要填充对齐时，Filler部分的字节会被插入到PF后来保证16 bit对齐。即如果WC是一个奇数，则CSI-2发送端会在PF后插入1字节的Packet Filler，保证PF是以16bit为单位结束的。
2. 插入额外的Filler字节，以保证每个C-PHY的每个lane都传输相同数量的16 bit字。

综上，对于两种形式的物理层实现，总结如下：

1. DI字段：均包括一个8bit的DI字段，由2 bit的VC和6 bit的DT构成。均包括VCX字段，用于扩展VC id的大小，但D-PHY的VCX为2bit，C-PHY为3bit。
2. WC字段：均包括一个16bit的WC字段，用来确定Packet Data的长度。注意两点：(1)WC的单位是字节数(2)只统计PH到PF之间的有效数据的长度。
3. ECC字段：对于D-PHY，6bit的ECC校验码用来校验PH中的DI+WC+VCX字段，能够纠正1bit错误并检测出2bit的错误。
对于C-PHY，没有ECC字段，这是由于在C-PHY物理层链路上如果发生了一个单符号错误会导致接收到的包头PH里有多bit错误出现，因此ECC字段不再有用。作为替代，CSI-2协议中对C-PHY使用一个16bit的CRC字段，用来校验RES+VCX+DT+WC，并将这些字段连同CRC字段一起复制多份发送出去，因此接收端可以发现1个或多个物理层链路上的传输错误并恢复。同时，C-PHY物理层会在PH中加入同步字，使得C-PHY接收端时钟信号也丢失的情况下恢复PH数据。
4. Data Payload：接收端均通过读取PH中的WC字段来判断Data Payload的长度，这个过程中接收端显然是无需寻找任何用作数据同步的编码的，因此Data Payload只需要保证自己以8bit为单位即可，对其内容没有任何限制。
5. CRC字段：接收端一旦接收完Data Payload后，均开始接收并校验PF中的checksum字段，同时自己计算也根据接收的数据计算checksum并比较，以判断传输过程中是否出现错误。
6. FILLER字段：只在C-PHY中使用，上文中已详尽叙述过。
### 短包结构
下图分别展示了D-PHY的短包结构和C-PHY的短包结构。无论采用哪种物理层结构，短帧结构相当于只有长帧结构中的PH，没有PF和中间的数据域。PH中依旧包括DI、WC、ECC三个部分，其中，DI和ECC含义与长帧结构中的相同，DI中依旧包括VC和DT，只不过DT只使用上面的Data Type表格中的0x00 ~ 0x0F这一范围。

短帧结构中的WC域用作短帧的数据域(Data Field)，具体含义为：如果该短帧的Data Type为帧同步(Frame Synchronization)，则数据域表示帧数；如果短帧的Data Type为行同步(Line Synchronization)，则数据域表示行数。
![image](https://github.com/user-attachments/assets/4ec6ca2a-3132-4849-a93d-20583f260039)
