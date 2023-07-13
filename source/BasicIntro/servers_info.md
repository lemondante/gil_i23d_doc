# 本地服务器

二三维重建组有一定数量的服务器用于测试分布式三维重建系统。配置为2019-2022年的主流水平辅以数个高性能工作站。

## 服务器位置信息

北大有线网的网段为162.105.x.x，在非自行构建内网时均为公网ip。实验室1326房间的子网为162.106.86.x，此屋内设备数量较多，故不能保证每台设备拥有固定ip。通常在大规模断电等突发意外后可能会造成ip变化，需要主动连接显示器查看新ip（也可能无法获得ip）。尽量不要长期关闭服务器防止ip被收回。交流363房间内入口侧网线的子网为162.105.101.x，设备数量少，ip充足。内侧（三层主机支架处）子网为162.105.98.x，ip充足，但常出现各种意外断电或断连现象，可能与交换机性能有关。陈毅松办公室内部休息室有六台运行*ContextCapture*的windows服务器。其无公网ip，均为路由器内网（192.168.x.x）。该房间内还有大量废弃服务器，可以询问王晓博老师获取。

后续不再赘述服务器位置，仅以子网作为区分。

## 服务器配置信息

二三维重建组主要的重建服务器如下（配置仅凭记忆，若有错误请及时修改）：

+ 86.72主服务器：两块28核E5，256G内存，GTX 1080ti，16TB硬盘。曾用ip为86.90，故早期wiki等文档内称为90服务器。是二三维重建组资料保存最为完整的服务器，通常能找到大部分历史存档。该服务器并行能力强，通常兼任i23d的Master服务器与Worker服务器。账号为gil-cpu，密码为zhuangfa777，曾经遭遇挖矿病毒，建议采用私钥登录。
+ 101.153、98.166：CPU为i7（9代左右），64G内存，RTX 2070，6TB硬盘。153服务器郑中天通常也有使用需求，故使用前须与其沟通。166可随意使用。两者通常均为Worker服务器。账密均为graphics1/2。
+ 98.15、98.16、98.17：CPU为i7（6-9代），64G内存，GTX 1060-RTX 2070，6TB硬盘。其均已备案并可在校外访问。备案具体信息询问雷静文老师。其上部署了东信云系统，也曾布置过i23d流水线，可自行尝试测试。硬盘内数据可能被重置过，机械硬盘可能尚未装载，需要自行mount。曾经作为worker服务器使用，但目前较少使用，建议询问江西林同学。东信云工作人员也可使用VPN访问这三台服务器。账号均为root，密码：15-graphics3，16-graphics5，17-graphics6。
+ 101.201：CPU为i7，32G内存，GTX 1080ti，6TB硬盘。此服务器通常跑轻量级任务，已配置anaconda环境。账密均为graphics4。
+ 陈毅松老师办公室服务器：六台Windows服务器可分为一台高性能主服务器（最右侧）与五台从服务器。主服务器CPU为两块志强银牌4215R处理器，512G内存，RTX3090显卡，2TB固态硬盘与16TB机械硬盘。主要用于*ContextCapture*在大规模数据集时的空三处理。其余五台CPU为i5-i7，内存为32G-64G，均含有固态硬盘，但部分系统未安装至固态盘上。通常可以用远程控制软件（如向日葵）控制主服务器，主服务器采用Windows自带的远程桌面控制访问从服务器，以达到访问所有服务器的效果。六台服务器账户密码均为graphics。
