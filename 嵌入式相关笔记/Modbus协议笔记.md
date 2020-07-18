MODBUS：应用层报文传输协议，用于在通过不同类型的总线或网络连接的设备之间的客户机/服务器通信。

PDU: 与基础通信层无关的简单协议数据单元
ADU: 应用数据单元
![image-20200717144357080](D:\Saber_Workshop\Personal\Doc\Markdown_Note\Modbus协议笔记.assets\image-20200717144357080.png)

**ASCII 模式和RTU模式：**
ASCII传输模式：消息帧以英文冒号(":",ASCII 3A Hex)开始，以回车和换号（CRLF，ASCII 0D and 0A Hex）符号结束，允许的传输的字符集为十六进制的0~9和A~F；网络中的从设备监视传输通路上是否有英文冒号（“：”），如果有的话，就对消息帧进行解码，查看消息中的地址是否与自己的地址相同，如果相同的话，就接收其中的数据；如果不同的话，则不予理会。该模式采用纵向冗余校验（Longitudinal Redundancy Check ，LRC)） 的方法来检验错误

![img](https://pic4.zhimg.com/80/v2-3940112714da28a6e3c6fde28ca6a502_720w.png)

RTU（Remote Terminal Unit）模式下，每个字节可以传输两个十六进制字符，比如十六进制数0xAF，直接以十六进制0xAF（二进制：10101111）进行发送，因此它的发送密度比ASCII模式高一倍；RTU模式采用循环冗余校验（CRC）

![img](https://pic3.zhimg.com/80/v2-1a154a830627feca91e5948fcf2dba2b_720w.png)



**数据编码**
使用‘big-Endian’表示地址和数据项。当发送多个字节时，首先发送最高有效位。



**MODBUS<font color=red>事务处理</font>的定义**

服务器侧：
![image-20200717150639488](D:\Saber_Workshop\Personal\Doc\Markdown_Note\Modbus协议笔记.assets\image-20200717150639488.png)





