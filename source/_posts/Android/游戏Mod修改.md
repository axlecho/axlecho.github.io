游戏Mod制作
===

本想写个明日方舟Mod教程，结果目标加壳了，算了算了

<!-- more -->
游戏Mod就是在原有的游戏上修改部分功能的修改版游戏，一般体现为修改攻击力，HP等
现在的游戏大部分基于Unity 3D的引擎开发，其逻辑实现全都集中在`libil2cpp.so`上，因此只需修改该库便可实现效果

准备阶段
---
那让我们开始吧

### 目标 
这里我们选[明日方舟](https://ak.hypergryph.com)这款游戏，因为弱联网的缘故，反破解手段用得比较少（是根本没有）
下载目标apk并解压出`libil2cpp.so`及`global-metadata.dat`
![明日方舟修改_目标](/images/明日方舟修改_目标.png)

### 工具准备
**Il2CppDumper**
用于反编译`global-metadata.dat`的符号表及生成`libil2cpp.so`的符号加载脚本
此工具为开源工具[github](https://github.com/Perfare/Il2CppDumper)

**IDA**
用于反汇编`libil2cpp.so`，主要的修改工作用于此

**jarsigner**
用于重新签名apk，安装JDK便可得到该工具

![明日方舟修改_工具](/images/明日方舟修改_工具.png)

反编译
---
打开**Il2CppDumper**，第一次选`libil2cpp.so`，
![明日方舟修改_反编译_il2CppDump_1](/images/明日方舟修改_反编译_il2CppDump_1.png)

再选`global-metadata.dat`
![明日方舟修改_反编译_il2CppDump_2](/images/明日方舟修改_反编译_il2CppDump_2.png)

模式我们选`Auto(Plus)`
![明日方舟修改_反编译_il2CppDump_3](/images/明日方舟修改_反编译_il2CppDump_3.png)