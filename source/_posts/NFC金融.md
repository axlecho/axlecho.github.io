---
title: NFC金融
date: 2019-01-22 11:45:21
tags: android_framework
---

最近两天研究了下移动金融的东西，水是相当的深。

NFC有三种模式：点对点(p2p)，读/写卡(reader/writer)，卡模拟(card emulation),在移动金融领域中一般只有卡模拟模式。

乱七八糟的卡
说到卡模拟，先要明白这里的卡指的是什么，这里的卡指的是集成电路卡(Integrated Circuit Card,IC卡),又称智能卡(Smart card),还有一些乱七八糟的叫法。
et.新的银行卡，工卡，公交卡等，基本都是这玩意了。

另外的还有身份识别卡（Identification Card），比较老的只读卡。
et.老的门禁卡，停车卡

还有一个很乱的概念**cpu卡,不清楚是什么的东西

##### ic卡
ic卡又分为接触式IC卡和非接触式IC卡，很形象。。没必要解释。。
接触式IC卡常见的有银行卡，插入atm应该就是接触式的。。所以银行卡具有接触式IC卡和非接触式IC卡(pos机)的两种功能。

ISO14443协议是非接触式IC卡标准(Contactless card standards)协议。

##### nfc卡
nfc卡，故名思意，就是基于nfc的ic卡，也就是作用距离在10厘米以内的ic卡。

1
近场通信（Near Field Communication,NFC）是一种短距高频的无线电技术，在13.56MHz频率运行于10厘米距离内。
nfc卡的种类，其实就是各个公司基于标准的实现

NfcA — ISO 14443-3A
NfcB — ISO 14443-3B
NfcF — JIS 6319-4
NfcV — ISO 15693
IsoDep — ISO 14443-4
M卡指的是Mifare卡，MIFARE Classic是NfcA ，MIFARE DESFire是IsoDep。

PS. 银行卡非接触式部分是基于IsoDep，接触式部分是基于ISO7816实现的
【NFC】 NfcA/NfcB/NfcF/NfcV/IsoDep/Ndef/Mifare/Felica/Pboc/ISOxxxx 都是些什么鸟玩意？

卡模拟
卡模拟到现在有两种实现方案，一种是基于软件的，被称为主机卡模式（Host-based Card Emulation，HCE），另外一种是基于硬件的（不知道叫什么）。
这里重点关注的是基于硬件的做法。既然是卡模拟，那么最大的问题就是安全的问题，所以有了SE出现。

##### SE
安全元件（Secure Element，SE），通常以芯片形式提供。为防止外部恶意解析攻击，保护数据安全，在芯片中具有加密/解密逻辑电路。

现在主要有三种形态

内置SE模式（eSE）
UICC（SIM卡）
AdvancedSecurity SD card（ASSD SD卡）

##### 模拟的卡（APPLET)
模拟的卡是一种程序，运行于SE中，也是用java开发，叫做java card applet
概念还是很模糊 – 比如所APPLET是怎么加载到SE上的

##### TSM
发卡服务器,APPLET由TSM来发送安装。。

SMARTCARD
EMV
PBOC
