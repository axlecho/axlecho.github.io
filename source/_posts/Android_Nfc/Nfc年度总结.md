---
title: Nfc年度总结
date: 2019-01-22 11:51:44
categories: Android_Nfc
tags: 
	- android
	- nfc
	- framework
---


NXP代码整合：[NXPNFCProject](https://github.com/NXPNFCProject)

framework: 定义Nfc数据类型等 – NXPNFCProject/NFC_NCIHAL_base
NfcNci: Nfc行为实现，开启关闭Nfc，Tag分发系统，beam等 – NXPNFCProject/NFC_NCIHAL_Nfc
libnfc-nci: Hal层实现 – NXPNFCProject/NFC_NCIHAL_libnfc-nci
firmware: NFC固件 – NXPNFCProject/NXPNFCC_FW
driver: NFC驱动 – NXPNFCProject/NXPNFC_I2CDriver

spi: framework，app，hal – NXPNFCProject/NXPNFC_P61_SPI_Services
driver – NXPNFCProject/NXPNFC_P61_SPI_Services