# CORE
[toc]

## 实验流程
**注**：可执行脚本都存放于`/usr/local/bin/`目录下
### `sudo core-daemon`
1. `sudo core-daemon`会启动CORE的守护进程
    (1). 实例化一个`CoreServer`（`from core.api.tlv import CoreServer, CoreUdpServer`）
    (2). 启动其他内容并初始化**grpc api**

2. 实例化[CoreServer](./api.md#coreserver.py)
    (1). 实例化`CoreEmu`(`from core.emulator import coreemu`)
    (2). 读取配置项
    (3). 配置服务器连接（**没看懂**）
3. 实例化[CoreEmu](./emulator.md#coreemu.py)
    (1). 修改权限，
    (2). 读取配置项
    (3). 创建**session**列表
    (4). 加载服务(import 和)
    (5). 配置服务，目前看来主要是初始化了一些后面配置服务所需要的类和参数。
    (6). 检查可执行文件
    (7). 监听关闭指令。

**日志配置文件**：`/etc/core/logging.conf`
**配置文件**：`/etc/core/core.conf`

### `core-gui`
* **此处为原版UI**
1. core-gui运行bash脚本以启动
    (1). -h参数可查看帮助
    (2). 设置临时环境变量`export ...`
    (3). 创建临时配置文件夹
    (4). 设置变量`core = $LIBDIR/core.tcl`为老版GUI中的的程序入口。
    (5). 定位`wish8.5 binaries`？？？
    (6). 创建`/home/user/.core`文件夹并检查读写权限。
    (7). `GUI config directory should not be a file (old prefs)`没看懂
    (8). 分析`core-gui`命令后面的参数并执行相应动作（调用core.tcl）。
    (9). `cd $CORE_START_DIR`（这个文件夹是当前用户文件夹）

**参考坐标配置位置：.core/pref.conf**


### `core-pygui`
1. core-pygui
    (1). 读取并分析参数
    (2). 加载图片
    (3). 实例化`Application`类，
    (4). 进入图形化界面主循环（不懂，但是猜测是持续监听UI的事件）
2. 实例化`Application`类

3. 进入mainloop（python 库中定义的函数）

**参考坐标配置位置：.coregui/config.yaml** ，也可以设置其他参数。

**如果要改变日志的级别**：
```bash
core-pygui -l LEVEL
core-pygui --level LEVEL
#LEVEL可以的取值有：DEBUG, INFO, choices=["DEBUG", "INFO", "WARNING", "ERROR", "CRITICAL"]
```


## NOTE
- `CoreNode`类中，创建 **net_client** 时有`use_ovs`参数。
- `CoreNodeBase`类中，定义了添加服务到待配置列表中和配置服务的项目。
- 节点创建中使用vnode创建了一个新的命名空间。

### 开始仿真操作
开始仿真的时间定义位于**core.gui.coreclient.py**中，
函数：`start_session`。
`core.gui.coreclient.py`中的`start_session`被调用后，会调用`core.api`中的客户端，然后客户端调用服务端，详见[core.api](./api.md)和[core.gui](./GUI-py.md)

### 节点startup
1. 创建节点的文件夹，检查节点是否已经创建（up）
2. 使用vnode创建一个新的命名空间。 并存储其进程号pid。
3. 创建vnode client
4. 启用容器的本地回环网络接口
5. 设置节点的hostname
6. 更改节点中的`self.up`属性为`True`
7. 创建节点私有文件夹（位于节点文件夹下）。


### 网络问题
节点的net_client有两种，根据`use_ovs`参数可创建LinuxNetClient和OvsNetClient(LinuxNetClinet)



## [node](./node.md#node)

## [services](./services.md#services)





## [configservice](./configservice.md#configservice)






## 程序




