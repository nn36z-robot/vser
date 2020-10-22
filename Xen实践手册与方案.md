# 基于Xen的服务器虚拟化实践手册及中学数据中心搭建推荐方案

>作者：
>
>1. 苏广（南宁市第三十六中学，信息中心，信息技术教师兼网络管理员，副高级职称）
>
>2. 韦维（南宁市第三十六中学，信息中心，信息技术教师，中级职称）
>3. 刘存宝（南宁市第三十六中学，信息中心，信息技术教师，中级职称）
>4. 申通（南宁市第三十六中学，语文教研组，语文教师，中级职称）
>5. 甘翊名（南宁还第三十六中学，信息中心，网络管理员，中级职称）



[toc]

## 1. 概述
   Xen 是思杰（Citrix）公司推出的一款开源的服务器虚拟化项目。它具有功能强大、简单易用和中文界面等诸多优势，能让人轻松上手，迅速掌握。 

   经过本课题组3年的实践证明，它是一款非常适合中学使用的服务器虚拟化平台。

   为此，特编写本手册供中学网络管理员参考。



### 1.1 理解基于Xen技术架构下的服务器虚拟化：

- 管理员不直接使用物理服务器。

- 物理服务器要安装Xen Server操作系统。

- 管理员所操作的是一个名为“Xen Center”的软件，由它对装有Xen Server操作系统的物理服务器进行管理。


```mermaid
graph LR
A[管理员] -->B[XenCenter]
B -->C[XenServer]
C -->D[物理服务器]
```

### 1.2  资源池和虚拟服务器

- Xen Center会将所有“同类”的物理服务器转化为资源“池”进行管理。

- 所在Xen技术架构下，物理服务器不是孤立的，所有的服务器硬件资源被建成一个个资源池。

- 资源“池”建成后，可以灵活地从各个资源“池”中创建出一台台虚拟服务器。

  


## 2. 硬件环境的准备

![硬件设备连接示意图](assets\硬件设备连接示意图-1601302902945.png)

### 2.1 服务器

Xen技术架构下，对服务器是有要求的：
- 同一个资源“池”所有的服务器CPU必须属于同一品牌。
- 所有的CPU支持虚拟化技术，并启用虚拟化功能。
- 所有CPU具有相同的功能集，否则不利于虚拟机实时迁移。

### 2.2 网络架构

注意以下几点：
- 物理服务器只安装Xen Server，不需要配过大的硬盘。
- 虚拟机的操作系统都在网络存储上划分出特定空间来安装。
- 服务器若有多个网络接口，所有接口最好都能接入网络。
- 不同虚拟机使用不同的网卡，避免过度负载。

### 2.3 网络存储
为了使用高可用（HA）功能，必须使用网络存储。

实在是没有专业的网络存储硬件设备的话，可以考虑用一台服务器来搭建网络存储，用以替代硬件。

Xen Server所支持的方案是NFS和iSCSI，下面介绍它们的安装方法（如果已经有存储硬件，可以忽略2.3.1和2.3.2两小节的内容）。

#### 2.3.1 安装NFS服务器的方法
NFS即网络文件系统，是当前主流异构平台共享文件系统之一，可以考虑用它来作为XenServer的网络存储解决方案。

假定某台服务器，已经安装了Ubuntu Linux操作系统，其IP地址设为192.168.0.103。

通过安装NFS服务，使其变成一台网络存储服务器，使得网络上访问该存储的地址为：192.168.0.103:/mnt/data。
具体操作如下：

1. 安装软件包
```
$ sudo apt-get install nfs-kernel-server
```
2. 将指定目录设为网盘
打开/ext/exports文件
```
$ sudo vim /etc/exports
```
如若要将/data/nfs目录让该服务器共享, 则在”/etc/exports“文件末尾添加如下语句
```
/data/nfs    *(rw,sync,no_root_squash)
```
更新exportfs文件
```
$ sudo exportfs -r
```

3. 重启nfs服务
```
 $ sudo /etc/init.d/nfs-kernel-server restart 
```

4. 将上面指定的/data/nfs目录挂载出来
```
$ sudo mount 192.168.0.103:/data/nfs /mnt/data
```


#### 2.3.2 安装iSCSI服务器的方法

假定某台服务器，已经安装了Ubuntu Linux操作系统。

1. 安装软件包
```
$ sudo apt-get install iscsitarget iscsitarget-source iscsitarget-dkms
```

2. 修改iscsitarget配置文件
```
 $sudo vim /etc/default/iscsitarget
```
在文件中修改一句，将原来的false改为true，如下：
```
ISCSITARGET_ENABLE=true
```

3. 创建500G的LUN0
```
$sudo dd if=/dev/zero of=/mnt/volume0/storlun0.bin count=0 obs=1 seek=500G
```

4. 配置IQN
```
$ sudo nano /etc/iet/ietd.conf
```
在文件最后添加
```
Target iqn.2017-08.local.mynet:storage.sys0
Lun 0 Path=/mnt/volume0/storlun0.bin,Type=fileio,ScsiId=lun0,ScsiSN=lun0
```

5.重新启动服务

```
$ sudo service iscsitarget restart
```

## 3.  相关软件的安装
物理服务器，以光盘或ISO形式安装XenServer操作系统。
一台管理PC上安装Windows操作系统，并安装Xen Center软件和汉化补丁。


### 3.1 XenServer的安装

本手册使用的是XenServer6.5版本的操作系统。

新版软件的下载地址：https://www.citrix.com/zh-cn/downloads/citrix-hypervisor/，安装和使用与本手册介绍的XenServer6.5一致，不做赘述。

下载 XenServer 7.0 Standard Edition，页面下载链接如下图所示：

![image-20200901220311929](assets\image-20200901220311929.png)

1. 将ISO文件刻录成光盘，在服务器上放入光盘，并启动光盘安装。

2. 光盘启动后，根据提示按回车（Enter）键继续，如下图所示。

   ![Server安装1](assets\Server安装1.jpg)

   按回车键后，会出现多行英文进度，这是在检查硬件和加载安装程序。

   ![Server安装2](assets\Server安装2.jpg)

3. 等待一会儿后，开始出现蓝色的安装向导。一开始和很多Linux系统的安装相似，选择键盘映射为美国键盘（us），防止后续操作因键盘映射导致错选。

   ![Server安装3](assets\Server安装3.jpg)

4. 由于安装XenServer要清除所有硬盘数据，这一步是让你确认是否继续安装（选择Ok），或者放弃安装并重启（选择Reboot）。

   ![Server安装4](assets\Server安装4.jpg)

5. XenServer是开源软件，用户使用的时候必须遵循它所规定的开源协议。这一步询问你是否接受“最终用户许可协议（EULA)”。

   ![Server安装5](assets\Server安装5.jpg)

6. 此处选择安装盘，Enable thin provisioning 选择后会使XenDesktop优化存储容量，一般没有必要，因为服务器的硬盘很少会使用。

   ![Server安装7](assets\Server安装7.jpg)

7. 选择安装来源，因为是刻录光盘后安装，这里直接选本地媒介（Local media）即可。

   ![Server安装8](assets\Server安装8.jpg)

8. 选择是否安装额外的包。

   ![Server安装9](assets\Server安装9-1599226794068.jpg)

9. 检查光盘的完整性，如果确认光盘没问题，则跳过验证（Skip verification） ，不确定光盘是否受损，则选择验证安装源（Verify install source）。

   ![Server安装10](assets\Server安装10.jpg)

10. 指定root账号密码，root是Linux最大权限用户，类似于Windows的administrator用户。

    ![Server安装11](assets\Server安装11.jpg)

11. 配置网络，包括IP地址和DNS。

    ![Server安装13](assets\Server安装13.jpg)

    ![Server安装14](assets\Server安装14.jpg)

12. 选择时区（上海）和时间服务器（NTP），如果不想指定则选择（Manual time entry）。

    ![Server安装15](assets\Server安装15.jpg)

    ![Server安装16](assets\Server安装16.jpg)

    ![Server安装17](assets\Server安装17.jpg)

13. 安装向导到此设置完毕，开始安装（Install XenServer）。

    ![Server安装19](assets\Server安装19.jpg)

    ![Server安装20](assets\Server安装20.jpg)

14. 安装成功后，取出光盘，重启电脑，即可进入到XenServer界面

    ![Server安装22](assets\Server安装22.jpg)

    ![Server安装24](assets\Server安装24.jpg)

    ![Server安装25](assets\Server安装25.jpg)

    服务器完全启动后，看见这个界面，说明XenServer已经安装成功。

    
### 3.2 XenCenter的安装
 		XenCenter是XenServer配套的管理维护软件。安装它需要采用一台Windows操作系统的普通PC。官方还提供了中文的界面补丁和中文的帮助文档，非常友好。值得注意的是，安装XenCenter需要.net framework的支持，因此可以根据提示进行安装相应版本的 .net framework安装包。



新版软件的下载地址：https://www.citrix.com/zh-cn/downloads/citrix-hypervisor/

下载XenCenter 7.0.1 Windows Manager Console，和语言包XenCenter 7.0.1 Localization Version。

页面下载链接如下图所示：

![image-20200901220558680](assets\image-20200901220558680.png)

下载的2个exe文件都安装后，即可运行XenCenter开始使用，它的软件界面如下：

![](assets\image-20200901215134390.png)

### 3.3  将服务器添加到XenCenter中管理
   在XenCenter上面的工具栏中选择“添加新服务器”按钮，会出现“添加新服务器”对话框，如下图所示：

![image-20200901222036455](assets\image-20200901222036455.png)

   在弹出的“添加新服务器”对话框中，下拉列表框“服务器”输入服务器IP地址，或者看看它能不能自动扫描出本网段内已经装好的XenServer服务器。选好（填好）后，并输入安装XenServer时输入的账号密码即可加入到主界面之中。如下图所示：

![image-20200901222331801](assets\image-20200901222331801.png)

成功添加服务器后，就可以在XenCenter中直接看到该服务器的基本信息，同时可以对服务器的“内存”、“存储”、“网络连接”、“NIC（网卡）”、“控制台”、“性能”、“用户”7大项目进行管理。如下图所示：

![image-20200901222637282](assets\image-20200901222637282.png)

同样的，把所有安装了XenServer的服务器都添加进来，如图：

![image-20200901222738678](assets\image-20200901222738678.png)



## 4. 新建池

在工具栏上选择“新建池”按钮，弹出“创建新池”的对话框，如下图：

![001](assets\001.jpg)

选择主服务器，并勾选其他服务器成员。

选择“创建池”按钮即可。

![003](assets\003.jpg)



## 5. 新建存储

在XenCenter左侧树形栏目中选择相应的池，然后在工具栏上选择“新建存储”按钮，将弹出“新建存储库 - 服务器池”的对话框。

### 5.1 NFS存储的建立方法

目前市面上很多网络存储柜都支持NFS。（如果没有存储柜硬件设备，可以2.3.1参考搭建NFS服务器。）

1. 选择“NFS VHD(V)"类型。

    ![001](assets\001-1599058171366.jpg)
2. 给存储库命名

    ![007](assets\007.jpg)
3. 正确填写NFS共享名称：

![006](assets\006.jpg)

   如果之前配置的NFS没有错误，就能顺利完成新建存储。

![008](assets\008.jpg)

![005](assets\005.jpg)

### 5.2 iSCSI和HBA存储的建立方法
1. 目前市面上很多网络存储柜都支持iSCSI。（如果没有存储柜硬件设备，可以2.3.2参考搭建iSCSI服务器。）

   ![iscsi连接](assets\iscsi连接.jpg)

   ![iscsi连接2](assets\iscsi连接2.jpg)

2. HBA的全称为Host Bus Adapter，即主机总线适配器。它是一种硬件适配卡，可连接多种存储器，由于没有这类硬件，未能进行实验。

## 6. 新建虚拟服务器
在XenCenter工具栏上单击“新建VM”按钮，会弹出“新建VM”向导，后续共有7个步骤。

### 6.1 选择VM模板。
   XenCenter自带很多不同的 虚拟机（VM） 模板， 每个模板都包含安装相应操作系统所需的最佳CPU、内存、存储和虚拟网络等设置信息。

![选择VM模板](assets\001-1600650978787.jpg)
    
### 6.2 为新虚拟机命名。
   可以根据自己的喜好命名，但通常最好使用描述性的名称。 虽然XenCenter 不会针对  VM 名称实施唯一性约束，但是建议您尽管避免重名。

### 6.3 查找操作系统安装介质。

![安装介质](assets\002-1600651045794.jpg)
   安装介质可以为：

   1. 从ISO库或DVD驱动器安装，此项可以选择ISO库中的ISO映像或者直接选择从池中某台物理服务器的DVD驱动器进行安装。

   2. 从网络启动，此选项适用于 Windows 的 PXE/网络引导和 Other install media（其他安装介质）模板。 

   3. 从URL安装，此选项适用于 CentOS、SUSE Linux Enterprise Server 和 Red Hat Linux 操作系统。选择从 URL 安装并输入 URL，URL 必须包含采用以下格式的服务器 IP 地址和存储库路径，例如：nfs://10.10.32.10/SLES10，其中 10.10.32.10 是 NFS 服务器的 IP，/SLES10 是安装存储库的位置。除了NFS之外， 常见的URL地址还有ftp和http的协议的URL。

### 6.4 选择主服务器。

![选择主服务器](assets\003-1600651207265.jpg)
   主服务器是在池中物理服务器。 为 VM 指定了主服务器后，如主服务器没有问题，XenServer 将会一直尝试在该服务器上启动 VM，如果此操作不可行，将自动选择同一池中的其他服务器。  

### 6.5 CPU和内存分配。

![CPU分配](assets\004.jpg)

![内存分配](assets\005-1600651548290.jpg)

   由于XenServer是“一虚多”模式，不支持“多虚一”。所以CPU和内存的划分理论上不大于池中最高性能的物理服务器。
   其中，vCPU的数量，实际是核心数量，例如：分配 4 个 vCPU 将显示为 4 个插槽，每个插槽 1 个核心。 

```
注：“一虚多”，指一台服务器将虚拟出多台服务器使用。“多虚一”，指一台服务器不能支撑某个业务时，可以用多台服务器虚拟出一台更强的服务器。
```

### 6.6 虚拟存储配置。
   类似于服务器至少有一个硬盘，虚拟机也应至少有一个虚拟磁盘。XenServer技术架构下， VM 最多可以有 7 个虚拟磁盘以及 1 个虚拟 CD-ROM。
```
建议硬盘都尽可能从网络存储上获取空间，以便部署高可用，避免服务器故障导致某个应用系统无法使用。
```
   以下是一个典型的虚拟存储配置方案：

1. 参考前文”6. 新建存储“，新建VM时，会自动从已经建好的资源池中选择网络存储。下图为：从一台名为“huawei”的网络存储服务器上划分出32GB作为虚拟机的一个硬盘，给当前创建的VM使用。32GB是前文“ 7.1 选择VM模板”，模板中的默认值。

![006](assets\006-1600652197542.jpg)

2. 可以通过“属性”按钮，编辑磁盘相应信息。

![007](assets\007-1600652221984.jpg)

3. 系统盘比较好的容量为100GB，此处设为100GB。

![009](assets\009.jpg)



### 6.7 虚拟网络配置。 
   默认情况下，一台VM默认可配 4 个虚拟网络接口。 如要配置 4 个以上接口，请在创建后转至 VM 的网络连接选项卡并从中添加接口。如果需要更改虚拟网卡的物理网络、MCA 地址或服务质量（QoS）优先级，请选择该虚拟网络接口，然后单击属性。

![网络配置属性](assets\网络配置属性-1601303332872.png)


### 6.8  Windows虚拟机




## 7. 新建ISO库

安装操作系统时，安装介质可以选择从ISO 库或DVD 驱动器安装，选择从DVD 驱动器安装，就要到相应的物理服务器上放入CD/DVD光盘进行安装，在服务器管理中极为不便。使用从网上下载下来的ISO文件进行系统安装最为快捷，但XenServer在添加ISO文件的时候并没有VMWare（另一个虚拟化软件平台）方便，XenServer加载ISO是要建立ISO库的。

XenServer的ISO库有2种选择：1、Windows文件共享（CIFS）；2、NFS ISO。

![001](assets\001-1601236732865.jpg)

### 7.1  Windows文件共享（CIFS）


### 7.2  NFS ISO

## 8. 安装XenServer Tools

XenServer Tools 可提供高速I/O 以实现更高的磁盘和网络性能，即使不安装XenServerTools，虚拟机也能运行，但是性能在很大程度上会被限制。例如：不能彻底关闭、重启或挂起一台虚拟机；不能迁移一台正在运行的虚拟机；动态调整vCPU数量需要重启才生效。

因此XenServer Tools 必须安装在每个 VM上，以使 VM 具有完全受支持的配置。

### 8.1 Windows安装
单击安装XenServer Tools，会挂载XenServerTools.ISO。之后会在VM 控制台上打开 XenServer Tools 安装向导。XenServer Tools需要Microsoft.NET Framework 4.0 或更高版本支持，如果虚拟机运行的是Windows 7或更低版本，则需要在安装XenServer Tools 之前先安装Microsoft .NET Framework 4.0组件。

### 8.2 Linux安装

在命令行输入挂载命令：
```
#mount /dev/cdrom /mnt
```
![001](assets\001.png)
切换到/mnt目录下，查看目录中是否有安装文件。

```
#cd /mnt
#ls
```
![002](assets\002.png)

切换到Linux 目录，安装。

```
#cd Linux
./install.sh
```

![003](assets\003.png)

![004](assets\004.png)

## 9. 复制虚拟机与快照

### 9.1 复制虚拟机
可以通过复制（克隆）现有VM 的方式创建新的VM，XenServer 使用“完整复制”和“快速隆”种机制来复制VM：
* 完整复制生成VM 磁盘的完整副本；虚拟机必须安装了XenServer Tools 且处于关机状态，XenServer会直接完整复制原虚拟机，并生成新UUID，附加到克隆出来的虚拟机上
* 快速克隆（写入时复制）仅将修改的数据块写入磁盘，使用硬件级别的克隆功能将现有VM 中的磁盘复制到新VM。只有采用文件作为后端的VM 才支持此模式。写入时复制旨在节省磁盘空间并实现快速克隆，但会略微减低正常的磁盘性能。

```
注：只能在同一个资源池中直接复制VM。要将VM复制到其他池中的服务器，需要导出VM，然后再将其导入目标服务器。
```

### 9.2 快照

快照功能在系统管理中非常方便，也非常重要。某台服务器安装好系统后，可以回到某个历史状态，比如临时要加装某个对系统有侵入性的软件（无法完全卸载），安装前做一个快照，装完后立即再做一个快照，每次要用该软件时都可以切换到有该软件的状态，使用上非常便利。

1. 虚拟机右侧的选项卡中，选择“快照”选项卡，单击“生成快照”按钮。
![系统快照](assets\系统快照.png)
2. 安装好系统，马上建立快照，右侧可以看到一个状态变化的图示。意为当前是”干净系统“。
![系统快照1](assets\系统快照1.png)
3. 进行某些安全配置，再做一次快照，可以看到多了一个“安全防护”快照。
![系统快照2](assets\系统快照2.png)
4. 还有一个非常方便的功能——从快照创建VM，可以迅速复制出一个安装了系统和应用软件的虚拟机副本。
![系统快照3](assets\系统快照3.png)

## 10. 高可用性

### 10.1 高可用性的必要性
在应用系统的指标中，“高可用性”（High Availability）表示”不停工“持续服务的特性。在中学中，有很多信息化系统都需要具备高可用性，比如：门禁系统，消费（一卡通）系统，备授课系统，在线考试系统。这些系统，基本不允许中断服务，否则很容易酿成安全事故、教学事故等重大问题。传统的服务器或多或少都担心出现硬件故障等短时间难以解决的问题，很难保证不停服务。

因此，为了保障关键应用提供7*24小时99.99%以上的持续服务保障，Xen的高可用性功能显得非常必要。


### 10.2 配置高可用

XenServer支持高可用性配置，其开设的条件是：1.必须使用网络存储；2.虚拟机可以在任意物理服务器上正常运行；3.虚拟机有足够的硬件资源（CPU和内存）运行。若您的服务器安装和配置都按本文前面所推荐的方式配置的话（使用同一批次物理服务器且使用网络存储），开启高可用是非常简单的一件事。如果不满足，则需要进行测试整改，以满足服使用高可用性的先决条件。

1. 开启高可用性：

![000](assets\000.png)

2. 配置向导：

![001](assets\001-1601689808885.jpg)

3. 检测信号SR：

   ![003](assets\003-1601690785592.png)

4. 高可用性计划：

   ![004](assets\004-1601690807401.png)

5. 完成：

   ![005](assets\005.png)

## 11. 常见故障

### 11.1 物理服务器崩溃



### 11.2 网络中断



### 11.3 网络存储损坏



## 12. 中学服务器虚拟化推荐方案
![生产环境设备示意图](assets\生产环境设备示意图.png)

